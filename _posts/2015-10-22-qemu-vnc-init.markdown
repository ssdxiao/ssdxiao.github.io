---

layout: post

title:  qemu中VNC流程详解+代码分析

date:   2015-10-22 11:10:01

categories: 虚拟化
tags: 
  - 虚拟化
---

在虚拟化场景下，对于VM的访问可以使用VNC等可视化工具来操作。VNC的原理其实很简单，
qemu会对每一个虚拟机模拟一块网卡，而VM的显示信息都会留在这个网卡的显存中。qemu启动一个
VNC server，这个server其实就一个定时器，以一定的频率默认是（30ms）从显存中拿出显示的信息，
然后，当有VNC client连接上以后，定期的发送给VNC client就可以了。


这里描述的只是简单的流程。其实传输一个画面的像素还是需要非常大量的网络带宽的。所以为了显示效果
VNC仅仅传输屏幕上变化的像素，以及在传输中启用了视频压缩算法。

接下来就从代码角度来分析下这一流程。


首先是vnc入口

在vl.c中的main函数对vnc注册了初始化函数vnc_init_func, 

{%highlight c%}
#ifdef CONFIG_VNC
    /* init remote displays */
    qemu_opts_foreach(qemu_find_opts("vnc"), vnc_init_func, NULL, 0);
    if (show_vnc_port) {
        printf("VNC server running on `%s'\n",
               vnc_display_local_addr("default"));
    }
#endif

{%endhighlight%}

ui/vnc.c中包含了vnc-server所需要的主要代码。
vnc_init_func直接调用了vnc真正的初始化函数，vnc_display_init和vnc_display_open，其实早期的qemu在vl.c中

是直接调用这2个函数的。

{%highlight c%}
int vnc_init_func(QemuOpts *opts, void *opaque)
{
    Error *local_err = NULL;
    char *id = (char *)qemu_opts_id(opts);

    assert(id);
    vnc_display_init(id);                                                                               

                                                             
    vnc_display_open(id, &local_err);
    if (local_err != NULL) {
        error_report("Failed to start VNC server on `%s': %s",
                     qemu_opt_get(opts, "display"),
                     error_get_pretty(local_err));
        error_free(local_err);
        exit(1);
    }
    return 0;
}

{%endhighlight%}


函数vnc_display_init在开始的时候主要是将启动线程，同时注册dcl_ops


{%highlight c%}
void vnc_display_init(const char *id)
{
     VncDisplay *vs;

    if (vnc_display_find(id) != NULL) {
        return;
    }
    vs = g_malloc0(sizeof(*vs));
 
    vs->id = strdup(id);
    QTAILQ_INSERT_TAIL(&vnc_displays, vs, next);

    vs->lsock = -1;
#ifdef CONFIG_VNC_WS
    vs->lwebsock = -1;
#endif
  
    QTAILQ_INIT(&vs->clients);
    vs->expires = TIME_MAX;

    if (keyboard_layout) {
        trace_vnc_key_map_init(keyboard_layout);
        vs->kbd_layout = init_keyboard_layout(name2keysym, keyboard_layout);
    } else {
        vs->kbd_layout = init_keyboard_layout(name2keysym, "en-us");
    }
  
    if (!vs->kbd_layout)
        exit(1);
  
    qemu_mutex_init(&vs->mutex);
    //启动一个线程，该线程用来将屏幕更新的数据发送给client
    vnc_start_worker_thread();
  
    vs->dcl.ops = &dcl_ops;
    register_displaychangelistener(&vs->dcl);
}

{%endhighlight%}


vnc_display_open做了参数校验，然后根据指定的形式启动了vnc server监听端口。
由于函数太大，这里截取最关键的部分，同时代码可以看到，默认端口是5900

{%highlight c%}
      vs->lsock = inet_listen_opts(sopts, 5900, errp);
          if (vs->lsock < 0) {
              goto fail;
          }
.....

          //注册listen监听函数
          qemu_set_fd_handler2(vs->lsock, NULL,                                                         

                      vnc_listen_regular_read, NULL, vs);

          

{%endhighlight%}


dcl_ops 定义了vnc的相关操作，

{%highlight c%}
static const DisplayChangeListenerOps dcl_ops = {
    .dpy_name             = "vnc",
    .dpy_refresh          = vnc_refresh,
    .dpy_gfx_copy         = vnc_dpy_copy,
    .dpy_gfx_update       = vnc_dpy_update,
    .dpy_gfx_switch       = vnc_dpy_switch,
    .dpy_gfx_check_format = qemu_pixman_check_format,
    .dpy_mouse_set        = vnc_mouse_set,
    .dpy_cursor_define    = vnc_dpy_cursor_define,
  };

{%endhighlight%}


好了，那么当vnc连接时，就会触发vnc_connect函数，主要是初始化连接的client的socket，
以及注册回调等。最关键的会调用vnc_init_state。

{%highlight c%}
  static void vnc_connect(VncDisplay *vd, int csock,
                          bool skipauth, bool websocket)
  {
      VncState *vs = g_malloc0(sizeof(VncState));
      int i;
  
      vs->csock = csock;
      vs->vd = vd;
  
      if (skipauth) {
      vs->auth = VNC_AUTH_NONE;
      vs->subauth = VNC_AUTH_INVALID;
      } else {
          if (websocket) {
              vs->auth = vd->ws_auth;
              vs->subauth = VNC_AUTH_INVALID;
          } else {
              vs->auth = vd->auth;
              vs->subauth = vd->subauth;
          }
      }
      VNC_DEBUG("Client sock=%d ws=%d auth=%d subauth=%d\n",
                csock, websocket, vs->auth, vs->subauth);
  
      vs->lossy_rect = g_malloc0(VNC_STAT_ROWS * sizeof (*vs->lossy_rect));
      for (i = 0; i < VNC_STAT_ROWS; ++i) {
          vs->lossy_rect[i] = g_malloc0(VNC_STAT_COLS * sizeof (uint8_t));
      }
  
      VNC_DEBUG("New client on socket %d\n", csock);
      update_displaychangelistener(&vd->dcl, VNC_REFRESH_INTERVAL_BASE);
      qemu_set_nonblock(vs->csock);
  #ifdef CONFIG_VNC_WS
      if (websocket) {
          vs->websocket = 1;
  #ifdef CONFIG_VNC_TLS
          if (vd->ws_tls) {
              qemu_set_fd_handler2(vs->csock, NULL, vncws_tls_handshake_io,
                                   NULL, vs);
          } else
  #endif /* CONFIG_VNC_TLS */
          {
              qemu_set_fd_handler2(vs->csock, NULL, vncws_handshake_read,
                                   NULL, vs);
          }
      } else
  #endif /* CONFIG_VNC_WS */
      {
          //注册了从client接受输入的回调，比如鼠标移动，键盘按键等。
          qemu_set_fd_handler2(vs->csock, NULL, vnc_client_read, NULL, vs);
      }
  
      vnc_client_cache_addr(vs);
      vnc_qmp_event(vs, QAPI_EVENT_VNC_CONNECTED);
      vnc_set_share_mode(vs, VNC_SHARE_MODE_CONNECTING);
  
  #ifdef CONFIG_VNC_WS
      if (!vs->websocket)
  #endif
      {
          
          vnc_init_state(vs);
      }
  
      if (vd->num_connecting > vd->connections_limit) {
          QTAILQ_FOREACH(vs, &vd->clients, next) {
              if (vs->share_mode == VNC_SHARE_MODE_CONNECTING) {
                  vnc_disconnect_start(vs);
                  return;
              }
          }
      }
  }
{%endhighlight%}


vnc_init_state函数会和client进行vnc协议的版本协商等。

{%highlight c%}
  void vnc_init_state(VncState *vs)
  {
      vs->initialized = true;
      VncDisplay *vd = vs->vd;
  
      vs->last_x = -1;
      vs->last_y = -1;
  
      vs->as.freq = 44100;
      vs->as.nchannels = 2;
      vs->as.fmt = AUD_FMT_S16;
      vs->as.endianness = 0;
  
      qemu_mutex_init(&vs->output_mutex);
      vs->bh = qemu_bh_new(vnc_jobs_bh, vs);
  
      QTAILQ_INSERT_TAIL(&vd->clients, vs, next);
  
      graphic_hw_update(vd->dcl.con);
  
      vnc_write(vs, "RFB 003.008\n", 12);
      vnc_flush(vs);
      vnc_read_when(vs, protocol_version, 12);
      reset_keys(vs);
      if (vs->vd->lock_key_sync)
          vs->led = qemu_add_led_event_handler(kbd_leds, vs);
  
      vs->mouse_mode_notifier.notify = check_pointer_type_change;
      qemu_add_mouse_mode_change_notifier(&vs->mouse_mode_notifier);
  
      /* vs might be free()ed here */
  }   
{%endhighlight%}

此时，会触发vnc_dpy_switch函数。早期版本是没有这个的，但是相同功能函数为vnc_dpy_resize
该函数主要初始化了一个空间server surface，这个就是用来缓存从显存读取出的屏幕像素变化。
guest surface则对应了每一个连接，自己需要得到刷新的像素。

{%highlight c%}
  static void vnc_dpy_switch(DisplayChangeListener *dcl,
                             DisplaySurface *surface)
  {
      VncDisplay *vd = container_of(dcl, VncDisplay, dcl);
      VncState *vs;
      int width, height;
  
      vnc_abort_display_jobs(vd);
  
      /* server surface */
      qemu_pixman_image_unref(vd->server);
      vd->ds = surface;

      width = MIN(VNC_MAX_WIDTH, ROUND_UP(surface_width(vd->ds),
                                          VNC_DIRTY_PIXELS_PER_BIT));
      height = MIN(VNC_MAX_HEIGHT, surface_height(vd->ds));
      //这里调用pixman库来初始化了server端需要对显存内容缓存的空间。[pixman]是一个显存压缩的库
      vd->server = pixman_image_create_bits(VNC_SERVER_FB_FORMAT,
                                            width, height, NULL, 0);
  
      /* guest surface */
  #if 0 /* FIXME */
      if (ds_get_bytes_per_pixel(ds) != vd->guest.ds->pf.bytes_per_pixel)
          console_color_init(ds);
  #endif
      qemu_pixman_image_unref(vd->guest.fb);
      vd->guest.fb = pixman_image_ref(surface->image);
      vd->guest.format = surface->format;
      memset(vd->guest.dirty, 0x00, sizeof(vd->guest.dirty));
      
      //设置需要更新的像素，第一次连接为全部像素
      vnc_set_area_dirty(vd->guest.dirty, width, height, 0, 0,
                         width, height);
  
      QTAILQ_FOREACH(vs, &vd->clients, next) {
          vnc_colordepth(vs);
          //将像素刷新到client
          vnc_desktop_resize(vs);
          if (vs->vd->cursor) {
              vnc_cursor_define(vs);
          }
          memset(vs->dirty, 0x00, sizeof(vs->dirty));
          vnc_set_area_dirty(vs->dirty, width, height, 0, 0,
                             width, height);
      }
  }
{%endhighlight%}

vnc_desktop_resize 可以看到比较简单，就是调用vnc_flush来进行数据的传输

{%highlight c%}
  static void vnc_desktop_resize(VncState *vs)
  {
      if (vs->csock == -1 || !vnc_has_feature(vs, VNC_FEATURE_RESIZE)) {
          return;
      }
      if (vs->client_width == pixman_image_get_width(vs->vd->server) &&
          vs->client_height == pixman_image_get_height(vs->vd->server)) {
          return;
      }
      vs->client_width = pixman_image_get_width(vs->vd->server);
      vs->client_height = pixman_image_get_height(vs->vd->server);
      vnc_lock_output(vs);
      vnc_write_u8(vs, VNC_MSG_SERVER_FRAMEBUFFER_UPDATE);
      vnc_write_u8(vs, 0);
      vnc_write_u16(vs, 1); /* number of rects */
      vnc_framebuffer_update(vs, 0, 0, vs->client_width, vs->client_height,
                             VNC_ENCODING_DESKTOPRESIZE);
      vnc_unlock_output(vs);
      vnc_flush(vs);
  }

{%endhighlight%}


{%highlight c%}
  void vnc_flush(VncState *vs)
  {
      vnc_lock_output(vs);
      if (vs->csock != -1 && (vs->output.offset
  #ifdef CONFIG_VNC_WS
                  || vs->ws_output.offset
  #endif
                  )) {
          vnc_client_write_locked(vs);
      }
      vnc_unlock_output(vs);
  }
{%endhighlight%}

此后，定期刷新client端就会调用vnc_refresh，主要通过vnc_refresh_server_surface来让显存和
server surface对比，然后取出变化的像素，并通过vnc_update_client建立job，插入到发送队列中

graphic_hw_update函数式从硬件中来进行显存更新。这里不展开来讲。

{%highlight c%}
  static void vnc_refresh(DisplayChangeListener *dcl)
  {
      VncDisplay *vd = container_of(dcl, VncDisplay, dcl);
      VncState *vs, *vn;
      int has_dirty, rects = 0;
  
      if (QTAILQ_EMPTY(&vd->clients)) {
          update_displaychangelistener(&vd->dcl, VNC_REFRESH_INTERVAL_MAX);
          return;
      }
  
      graphic_hw_update(vd->dcl.con);
  
      if (vnc_trylock_display(vd)) {
          update_displaychangelistener(&vd->dcl, VNC_REFRESH_INTERVAL_BASE);
          return;
      }
  
      has_dirty = vnc_refresh_server_surface(vd);
      vnc_unlock_display(vd);
  
      QTAILQ_FOREACH_SAFE(vs, &vd->clients, next, vn) {
          rects += vnc_update_client(vs, has_dirty, false);
          /* vs might be free()ed here */
      }
  
      if (has_dirty && rects) {
          vd->dcl.update_interval /= 2;
          if (vd->dcl.update_interval < VNC_REFRESH_INTERVAL_BASE) {
              vd->dcl.update_interval = VNC_REFRESH_INTERVAL_BASE;
          }
      } else {
          vd->dcl.update_interval += VNC_REFRESH_INTERVAL_INC;
          if (vd->dcl.update_interval > VNC_REFRESH_INTERVAL_MAX) {
              vd->dcl.update_interval = VNC_REFRESH_INTERVAL_MAX;
          }
      }
  }
{%endhighlight%}

vnc_refresh_server_surface是对像素进行对比。

{%highlight c%}
  static int vnc_refresh_server_surface(VncDisplay *vd)
  {
      int width = MIN(pixman_image_get_width(vd->guest.fb),
                      pixman_image_get_width(vd->server));
      int height = MIN(pixman_image_get_height(vd->guest.fb),
                       pixman_image_get_height(vd->server));
      int cmp_bytes, server_stride, min_stride, guest_stride, y = 0;
      uint8_t *guest_row0 = NULL, *server_row0;
      VncState *vs;
      int has_dirty = 0;
      pixman_image_t *tmpbuf = NULL;
  
      struct timeval tv = { 0, 0 };
  
      if (!vd->non_adaptive) {
          gettimeofday(&tv, NULL);
          has_dirty = vnc_update_stats(vd, &tv);
      }
  
      /*
       * Walk through the guest dirty map.
       * Check and copy modified bits from guest to server surface.
       * Update server dirty map.
       */
      server_row0 = (uint8_t *)pixman_image_get_data(vd->server);
      server_stride = guest_stride = pixman_image_get_stride(vd->server);
      cmp_bytes = MIN(VNC_DIRTY_PIXELS_PER_BIT * VNC_SERVER_FB_BYTES,
                      server_stride);
      if (vd->guest.format != VNC_SERVER_FB_FORMAT) {
          int width = pixman_image_get_width(vd->server);
          tmpbuf = qemu_pixman_linebuf_create(VNC_SERVER_FB_FORMAT, width);
      } else {
          guest_row0 = (uint8_t *)pixman_image_get_data(vd->guest.fb);
          guest_stride = pixman_image_get_stride(vd->guest.fb);
      }
      min_stride = MIN(server_stride, guest_stride);
  
      for (;;) {
          int x;
          uint8_t *guest_ptr, *server_ptr;
          unsigned long offset = find_next_bit((unsigned long *) &vd->guest.dirty,
                                               height * VNC_DIRTY_BPL(&vd->guest),
                                               y * VNC_DIRTY_BPL(&vd->guest));
          if (offset == height * VNC_DIRTY_BPL(&vd->guest)) {
              /* no more dirty bits */
              break;
          }
          y = offset / VNC_DIRTY_BPL(&vd->guest);
          x = offset % VNC_DIRTY_BPL(&vd->guest);
  
          server_ptr = server_row0 + y * server_stride + x * cmp_bytes;
  
          if (vd->guest.format != VNC_SERVER_FB_FORMAT) {
              qemu_pixman_linebuf_fill(tmpbuf, vd->guest.fb, width, 0, y);
              guest_ptr = (uint8_t *)pixman_image_get_data(tmpbuf);
          } else {
              guest_ptr = guest_row0 + y * guest_stride;
          }
          guest_ptr += x * cmp_bytes;
  
          for (; x < DIV_ROUND_UP(width, VNC_DIRTY_PIXELS_PER_BIT);
               x++, guest_ptr += cmp_bytes, server_ptr += cmp_bytes) {
              int _cmp_bytes = cmp_bytes;
              if (!test_and_clear_bit(x, vd->guest.dirty[y])) {
                  continue;
              }
              if ((x + 1) * cmp_bytes > min_stride) {
                  _cmp_bytes = min_stride - x * cmp_bytes;
              }
              if (memcmp(server_ptr, guest_ptr, _cmp_bytes) == 0) {
                  continue;
              }
              memcpy(server_ptr, guest_ptr, _cmp_bytes);
              if (!vd->non_adaptive) {
                  vnc_rect_updated(vd, x * VNC_DIRTY_PIXELS_PER_BIT,
                                   y, &tv);
              }
              QTAILQ_FOREACH(vs, &vd->clients, next) {
                  set_bit(x, vs->dirty[y]);
              }
              has_dirty++;
          }
  
          y++;
      }
      qemu_pixman_image_unref(tmpbuf);
      return has_dirty;
  }
{%endhighlight%}

vnc_update_client，开始建立一个job，并将需要更新的像素放在job中，然后放入队列中。

{%highlight c%}
  static int vnc_update_client(VncState *vs, int has_dirty, bool sync)
  {
      vs->has_dirty += has_dirty;
      if (vs->need_update && vs->csock != -1) {
          VncDisplay *vd = vs->vd;
          VncJob *job;
          int y;
          int height, width;
          int n = 0;
  
          if (vs->output.offset && !vs->audio_cap && !vs->force_update)
              /* kernel send buffers are full -> drop frames to throttle */
              return 0;
  
          if (!vs->has_dirty && !vs->audio_cap && !vs->force_update)
              return 0;
  
          /*
           * Send screen updates to the vnc client using the server
           * surface and server dirty map.  guest surface updates
           * happening in parallel don't disturb us, the next pass will
           * send them to the client.
           */
          job = vnc_job_new(vs);
  
          height = pixman_image_get_height(vd->server);
          width = pixman_image_get_width(vd->server);
  
          y = 0;
          for (;;) {
              int x, h;
              unsigned long x2;
              unsigned long offset = find_next_bit((unsigned long *) &vs->dirty,
                                                   height * VNC_DIRTY_BPL(vs),
                                                   y * VNC_DIRTY_BPL(vs));
              if (offset == height * VNC_DIRTY_BPL(vs)) {
                  /* no more dirty bits */
                  break;
              }
              y = offset / VNC_DIRTY_BPL(vs);
              x = offset % VNC_DIRTY_BPL(vs);
              x2 = find_next_zero_bit((unsigned long *) &vs->dirty[y],
                                      VNC_DIRTY_BPL(vs), x);
              bitmap_clear(vs->dirty[y], x, x2 - x);
              h = find_and_clear_dirty_height(vs, y, x, x2, height);
              x2 = MIN(x2, width / VNC_DIRTY_PIXELS_PER_BIT);
              if (x2 > x) {
                  n += vnc_job_add_rect(job, x * VNC_DIRTY_PIXELS_PER_BIT, y,
                                        (x2 - x) * VNC_DIRTY_PIXELS_PER_BIT, h);
              }
              if (!x && x2 == width / VNC_DIRTY_PIXELS_PER_BIT) {
                  y += h;
                  if (y == height) {
                      break;
                  }
              }
          }
  
          vnc_job_push(job);
          if (sync) {
              vnc_jobs_join(vs);
          }
          vs->force_update = 0;
          vs->has_dirty = 0;
          return n;
      }
  
      if (vs->csock == -1) {
          vnc_disconnect_finish(vs);
      } else if (sync) {
          vnc_jobs_join(vs);
      }
  
      return 0;
  }
{%endhighlight%}
在前面vnc初始化时，初始化了一个线程，这个线程就会从队列中取出一个job，然后发送给client

该线程函数就是vnc_worker_thread_loop 
{%highlight c%}
  static int vnc_worker_thread_loop(VncJobQueue *queue)
  {
      VncJob *job;
      VncRectEntry *entry, *tmp;
      VncState vs;
      int n_rectangles;
      int saved_offset;
  
      vnc_lock_queue(queue);
      while (QTAILQ_EMPTY(&queue->jobs) && !queue->exit) {
          qemu_cond_wait(&queue->cond, &queue->mutex);
      }
      /* Here job can only be NULL if queue->exit is true */
      job = QTAILQ_FIRST(&queue->jobs);
      vnc_unlock_queue(queue);
  
      if (queue->exit) {
          return -1;
      }
  
      vnc_lock_output(job->vs);
      if (job->vs->csock == -1 || job->vs->abort == true) {
          vnc_unlock_output(job->vs);
          goto disconnected;
      }
      vnc_unlock_output(job->vs);
  
      /* Make a local copy of vs and switch output buffers */
      vnc_async_encoding_start(job->vs, &vs);
  
      /* Start sending rectangles */
      n_rectangles = 0;
      vnc_write_u8(&vs, VNC_MSG_SERVER_FRAMEBUFFER_UPDATE);
      vnc_write_u8(&vs, 0);
      saved_offset = vs.output.offset;
      vnc_write_u16(&vs, 0);
  
      vnc_lock_display(job->vs->vd);
      QLIST_FOREACH_SAFE(entry, &job->rectangles, next, tmp) {
          int n;
  
          if (job->vs->csock == -1) {
              vnc_unlock_display(job->vs->vd);
              /* Copy persistent encoding data */
              vnc_async_encoding_end(job->vs, &vs);
              goto disconnected;
          }
  
          n = vnc_send_framebuffer_update(&vs, entry->rect.x, entry->rect.y,
                                          entry->rect.w, entry->rect.h);
  
          if (n >= 0) {
              n_rectangles += n;
          }
          g_free(entry);
      }
      vnc_unlock_display(job->vs->vd);
  
      /* Put n_rectangles at the beginning of the message */
      vs.output.buffer[saved_offset] = (n_rectangles >> 8) & 0xFF;
      vs.output.buffer[saved_offset + 1] = n_rectangles & 0xFF;
  
      vnc_lock_output(job->vs);
      if (job->vs->csock != -1) {
          buffer_reserve(&job->vs->jobs_buffer, vs.output.offset);
          buffer_append(&job->vs->jobs_buffer, vs.output.buffer,
                        vs.output.offset);
          /* Copy persistent encoding data */
          vnc_async_encoding_end(job->vs, &vs);
  
      qemu_bh_schedule(job->vs->bh);
      }  else {
          /* Copy persistent encoding data */
          vnc_async_encoding_end(job->vs, &vs);
      }
      vnc_unlock_output(job->vs);
  
  disconnected:
      vnc_lock_queue(queue);
      QTAILQ_REMOVE(&queue->jobs, job, next);
      vnc_unlock_queue(queue);
      qemu_cond_broadcast(&queue->cond);
      g_free(job);
      return 0;
  }
{%endhighlight%}


[pixman]:http://www.pixman.org/
