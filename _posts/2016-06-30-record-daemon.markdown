---

layout: post

title:  html5录音程序
date:   2016-6-30 14:10:00
categories: html
tags: 
  - html
---

最近遇到一个需求，需要从网页上对声音进行录制并上传，因此研究了一下html的录音机制。
这里要用到已有的js库 recorder.js
下面把我写的html文件贴上来

{%highlight c%}
<!DOCTYPE HTML>
<html lang="en">
    <head>
        <meta charset = "utf-8"/>
        <title>Chat by Web Sockets</title>
        <script type="text/javascript" src="js/recorder.js"> </script>
        <script type="text/javascript" src="js/jquery.min.js"> </script>
         
        <style type='text/css'>
            
        </style>
    </head>
    <body>
        <audio  id="audio" controls="controls"  ></audio>
        
       <input type="button" id="record" value="Record">
       <input type="button" id="export" value="Export">
       <div id="message"></div>
    </body>
     
    <script type='text/javascript'>
            var onFail = function(e) {
                console.log('Rejected!', e);
            };
         
            var onSuccess = function(s) {
                var context = new AudioContext();
                var mediaStreamSource = context.createMediaStreamSource(s);
                rec = new Recorder(mediaStreamSource);
                //rec.record();
         
                // audio loopback
                // mediaStreamSource.connect(context.destination);
            }
         
            //window.URL = URL || window.URL || window.webkitURL;
            navigator.getUserMedia  = navigator.getUserMedia || navigator.webkitGetUserMedia || navigator.mozGetUserMedia || navigator.msGetUserMedia;
         
            var rec;
            //var audio = document.querySelector('#audio');
            var audio = $('#audio')[0];
         
            function startRecording() {
                if (navigator.getUserMedia) {
                    navigator.getUserMedia({audio: true}, onSuccess, onFail);
                } else {
                    console.log('navigator.getUserMedia not present');
                }
            }
            startRecording();
            //--------------------      
            $('#record').click(function() {
                rec.record();
                audio.src = "";
               var dd = ws.send("start");
                $("#message").text("Click export to stop recording");
     
                // export a wav every second, so we can send it using websockets
                intervalKey = setInterval(function() {
                    rec.exportWAV(function(blob) {
                         
                        rec.clear();
                        ws.send(blob);
                        //audio.src = URL.createObjectURL(blob);
                    });
                }, 3000);
            });
             
            $('#export').click(function() {
                // first send the stop command
                rec.stop();
                ws.send("stop");
                clearInterval(intervalKey);
                 
                ws.send("analyze");
                $("#message").text("");
            });
             
            var ws = new WebSocket("wss://" + window.location.host + "/record");
            ws.onopen = function () {
                console.log("Openened connection to websocket");
            };
            ws.onclose = function (){
                 console.log("Close connection to websocket");
            }
            ws.onmessage = function(e) {                
                console.log(e.data)
                audio.src = e.data;
            }
             
            
        </script>
</html>

{%endhighlight%}

网页很简单，就是一个录音功能，可以看下如何使用recorder.js的api。
这里用到了websocket来进行录音的上传。当然这不是唯一的方式。
需要说明一下，就是websocket在高版本chrome支持中需要使用ssl加密方式来访问，否则会出错。

网页中对录音程序，每3s进行一次截取，然后上传到服务器，在服务器端需要对音频文件进行合并处理。

这里说下我遇到的坑，并不是把每次上传的音频文件dd到同一文件就算合并了。而是要用专业工具进行音频剪辑。

这里我是使用python作为后台服务器。python中有moviepy这个库来专门做音视频剪辑，非常好用。
这里也把我的后台代码贴上来。
{%highlight c%}

#!/usr/bin/env python
# -*- coding: utf-8 -*-

from functools import partial
import threading
import tornado.httpserver
import tornado.websocket
import tornado.ioloop
import tornado.web
import os
import sys
import time
from datetime import datetime
from moviepy.editor import AudioFileClip, concatenate_audioclips
try:
    import simplejson as json
except ImportError:
    import json
import shutil

LISTENERS = []
AUDIO_PATH = "/tmp/audio"

class RealtimeHandler(tornado.websocket.WebSocketHandler):

    def check_origin(self, origin):
        return True

    def open(self):
        print "open"
        LISTENERS.append(self)

    def on_message(self, message):
        
        if message.startswith("start"):
            print "now start"
            self.current = "start"
            if os.path.exists(AUDIO_PATH):
                shutil.rmtree(AUDIO_PATH)
            os.mkdir(AUDIO_PATH) 
            self.i = 0
        elif message.startswith("stop"):
            print "now close"
            self.current = "stop"
            self.clips()
            if os.path.exists(AUDIO_PATH):
                shutil.rmtree(AUDIO_PATH)
        elif message.startswith("analyze"):
            pass
        else :
            self.save(message)

    def clips(self):
        print "clips start"
        filelist = os.listdir(AUDIO_PATH)
        audiolist=[]
        if filelist == []:
            print "no audio file find"
            return
        print filelist
        for file_temp in filelist:
             audiolist.append(AudioFileClip("%s/%s"%(AUDIO_PATH,file_temp)))
       
        final_clip = concatenate_audioclips(audiolist)
        final_clip.write_audiofile("static/abc.wav")
        self.write_message("/static/abc.wav")
        print "clips end"

    def save(self, message):
        try:
            self.i += 1;
            self.File = open("%s/temp%d.wav"%(AUDIO_PATH,self.i), "w")
            self.File.write(message)
        finally:
            self.File.close()

    def on_close(self):
        print "close"
        LISTENERS.remove(self)


settings = {
    "static_path": os.path.join(os.path.dirname(__file__), "static"),
    'auto_reload': True,
    }

application = tornado.web.Application(
    [('/record',RealtimeHandler),],
    **settings)


#usage from http://stackoverflow.com/questions/8045698/https-python-client
#openssl genrsa -out privatekey.pem 2048
#openssl req -new -key privatekey.pem -out certrequest.csr
#openssl x509 -req -in certrequest.csr -signkey privatekey.pem -out certificate.pem
http_server = tornado.httpserver.HTTPServer(
            application,
            ssl_options={
                "certfile": os.path.join("./", "certificate.pem"),
                "keyfile": os.path.join("./", "privatekey.pem"),
                }
        )
http_server.listen(443)
tornado.ioloop.IOLoop.instance().start()


{%endhighlight%}

好了 这样就可以进行网页录音，并播放录音文件了。

详细例子请参考 https://github.com/ssdxiao/audio

