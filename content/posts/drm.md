---
title: "drm介绍"
date: 2023-11-01T13:19:31+08:00
draft: false
authors: ["zzxyb"]
tags: ["drm"]
---

## 介绍
随着Linux图形发展的需求增长，需要更多的图形功能和性能，这促成了Direct Rending Infastructure（DRI）和Direct Rendering Manager（DRM）的出现，DRM是一个内核级别的子系统，提供了对图形设备的管理和访问控制，它允许用户空间的图形库和应用程序直接访问GPU进行渲染，而无需操作系统介入，它的主要作用如下：
- 图形硬件管理：DRM负责管理图形硬件资源，如显存、显卡以及显示设备。它允许内核与这些硬件设备进行通信和控制。
- 图形加速：提供对硬件加速功能的支持，例如3D渲染和视频加速。这使得图形渲染更加高效和流畅。
- 多显示器支持：允许多个显示器的管理和配置，包括扩展桌面、镜像模式等。
- 用户空间接口：提供了用户空间图形库（如Mesa 3D等）和应用程序与内核中DRM子系统进行通信的接口。
- 支持不同的图形协议：DRM在支持X Window System（X11）的同时，也能与新一代图形协议，如Wayland，进行整合。

这种更好的性能、3D加速和对硬件的更直接访问使得它可以支持X Window System和Wayland的图形输出和管理。其现状如下：
- 持续发展：随着硬件技术和图形需求的不断演进，DRM在Linux内核中也在不断发展。新的功能和改进不断加入，以满足新一代图形硬件和应用程序的需求。
- 多厂商支持：DRM的发展得到了多家硬件厂商的支持，这包括AMD、Intel、NVIDIA等，它们为Linux内核开发并贡献了各自硬件的DRM驱动程序。
- 支持新技术：DRM也在逐步支持新的图形技术，例如，对于低功耗图形处理的优化、支持更高分辨率、HDR显示以及机器学习和AI加速等方面的改进。
- Wayland和DRM集成：Wayland作为新一代图形显示协议，与DRM更为紧密地整合，提供更为直接和高效的图形渲染和显示方式。

总体而言，现如今很多Linux发行版上图形服务器后端都是对接的drm实现图形输出和相关控制，DRM在Linux图形子系统中扮演着关键角色，随着技术和需求的不断发展，它持续进化以适应新的硬件和应用场景，为Linux系统提供了强大的图形渲染支持。
{{< figure src="drm-timeline.svg" title="图1. DRM时间线" >}}

## DRM剖析
### Framebuffer && DRM
在谈drm之前必须先了解一下它的前辈Framebuffer，Framebuffer是指一块内存区域，用户存储显示设备上每个像素的颜色信息。在早期的计算机系统中，操作系统直接将图形数据写入这块显存区域，这被称为直接显存访问（Direct Framebuffer Access），这种方式简单、直接，但随着图形复杂程度增加和硬件发展，它变得难以满足高分辨率和复杂图形的需求，随着事件的推移，由DRM提供了更多先进的功能，它追加能替代了Framebuffer，并称为linux图形系统的主要组成部分（图形框架演变如下图2）。然而Frambuffer仍然在某些特定场景下有其用户之地，尤其是在一些嵌入式系统和特定的硬件上。
总的来说，Framebuffer到DRM的发展代表了图形处理在Linux系统中的进步和演进，从最初的直接访问到高级的硬件加速和更加强大的图形功能。drm为显示硬件的适配和图形的发展贡献了很大的力量。
{{< figure src="kms-vs-fbdev-display.svg" title="图2. Framebuffer到DRM" >}}

### drm-plane
drm_plane本质是对显示控制器中scanout硬件的抽象。简单来说，给定一个plane，可以让其与一个framebuffer关联表示进行scanout的数据，同时控制scanout时进行的额外操作，比如colorspace的改变，旋转、拉伸、偏移等操作。drm_plane是与硬件强相关的，显示控制器支持的plane是固定的，其支持的功能也是由硬件决定的。所有的drm_plane必为三种类型之一：
- Primary - 主plane，一般控制整个显示器的输出。CRTC必须要有一个这样的plane。
- Curosr - 表示鼠标光标图层（可选），一般启用的话称其为开启硬件光标，代码如下，流程如下图3。
- Overlay - 叠加plane，可以在主plane上叠加一层输出，可选。

```c
....
if (cursor != NULL && drm_connector_is_cursor_visible(conn)) {
	struct wlr_drm_fb *cursor_fb = get_next_cursor_fb(conn);
	if (cursor_fb == NULL) {
		wlr_drm_conn_log(conn, WLR_DEBUG, "Failed to acquire cursor FB");
		return false;
	}

	drmModeFB *drm_fb = drmModeGetFB(drm->fd, cursor_fb->id);
	if (drm_fb == NULL) {
		wlr_drm_conn_log_errno(conn, WLR_DEBUG, "Failed to get cursor "
			"BO handle: drmModeGetFB failed");
		return false;
	}
	uint32_t cursor_handle = drm_fb->handle;
	uint32_t cursor_width = drm_fb->width;
	uint32_t cursor_height = drm_fb->height;
	drmModeFreeFB(drm_fb);
    // 设置硬件光标，更新光标图像buffer
	int ret = drmModeSetCursor(drm->fd, crtc->id, cursor_handle,
			cursor_width, cursor_height);
	int set_cursor_errno = errno;
	if (drmCloseBufferHandle(drm->fd, cursor_handle) != 0) {
		wlr_log_errno(WLR_ERROR, "drmCloseBufferHandle failed");
	}
	if (ret != 0) {
		wlr_drm_conn_log(conn, WLR_DEBUG, "drmModeSetCursor failed: %s",
			strerror(set_cursor_errno));
		return false;
	}

    // 移动硬件光标位置
	if (drmModeMoveCursor(drm->fd,
			crtc->id, conn->cursor_x, conn->cursor_y) != 0) {
		wlr_drm_conn_log_errno(conn, WLR_ERROR, "drmModeMoveCursor failed");
		return false;
	}
} else {
	if (drmModeSetCursor(drm->fd, crtc->id, 0, 0, 0)) {
		wlr_drm_conn_log_errno(conn, WLR_DEBUG, "drmModeSetCursor failed");
		return false;
	}
}
....
```

{{< figure src="kms-display-plane.svg" title="图3. drm-plane框架图" >}}

{{< figure src="kms-plane.svg" title="图4. 多drm-plane合成流程图" >}}

在笔者看来，drm-plane与layer的概念类似，只是更加偏向硬件底层概念，开发者可以通过控制显示相关的参数将plane投显到显示器上任何区域（参考图5），这种映射机制提供了非常方便地偏移、缩放方法，如果在这一层上实现移动和缩放等动画，效率将会比使用OpenGL等API制作高，将频繁变化的图像抽象工作在一个单独的drm-plane也可以达到提高性能和省电的目的。
| 名字       | 描述  |
| ---------- | ----- |
| SRC_X | 当前framebuffer crop区域的起始偏移x坐标 |
|SRC_Y	| 当前framebuffer crop区域的起始偏移y坐标 |
|SRC_W	| 当前framebuffer crop区域的宽度         |
|SRC_H	| 当前framebuffer crop区域的高度         |
|CRTC_X	| 屏幕显示区域的起始偏移x坐标              |
|CRTC_Y	| 屏幕显示区域的起始偏移y坐标              |
|CRTC_W	| 屏幕显示区域的宽度                      |
|CRTC_H	| 屏幕显示区域的高度                      |

<p align="left">表1. drm-plane部分属性表</p>

{{< figure src="plane-update-coordinates.png" title="图5. drm-plane送显参数控制" >}}

### crtc（Cathode Ray Tube Controller）
CRTC 是显示控制器中的一个重要部分，主要用于管理显示设备的扫描和刷新。CRTC 负责生成视频信号的定时和同步，控制屏幕上像素的扫描和刷新，以确保正确的图像显示。它管理像素的输出到屏幕上的确切位置，以及刷新率、分辨率等显示参数。在现代的图形处理中，CRTC通常由图形处理单元（GPU）或显示控制器中的专用部分来控制，以确保正确的图像输出。

{{< figure src="kms-display-crtc.svg" title="图6. crtc框架图" >}}

drm legacy链路通过调用drmModePageFlip请求crtc更新显示器图像，其代码如下，流程如图6所示：
```c
static bool legacy_crtc_commit(struct wlr_drm_connector *conn,
		const struct wlr_drm_connector_state *state,
		uint32_t flags, bool test_only) {
	....

	if (flags & DRM_MODE_PAGE_FLIP_EVENT) {
		uint32_t page_flags = DRM_MODE_PAGE_FLIP_EVENT;
		if (flags & DRM_MODE_PAGE_FLIP_ASYNC) {
			page_flags |= DRM_MODE_PAGE_FLIP_ASYNC;
		}

		if (drmModePageFlip(drm->fd, crtc->id, fb_id,
				page_flags, drm)) {
			wlr_drm_conn_log_errno(conn, WLR_ERROR, "drmModePageFlip failed");
			return false;
		}
	}

	return true;
}
```

{{< figure src="page-flipping.svg" title="图7. buffer更新流程图" >}}

### encoder
编码器从CRTC获取像素数据，并将其转换为适合任何连接器的格式。在某些设备上，CRTC可以将数据发送到多个编码器。在这种情况下，两个编码器将从同一扫描输出缓冲区接收数据，从而在连接到每个编码器的连接器上产生“克隆”显示配置。
在Linux系统中，DRM（Direct Rendering Manager）是用于处理图形显示的子系统，负责在用户空间和图形硬件之间提供接口。DRM编码器在Linux中扮演了关键角色，主要用于对图形和视频内容进行编码、解码、处理和显示。在DRM框架中，编码器的作用包括但不限于：
- 视频压缩和解压缩： 编码器负责对视频内容进行压缩，以减小数据量并更有效地传输和存储视频流。解码器用于解压缩视频内容，以便图形硬件能够正确地渲染和显示内容。
- 图形处理和渲染： DRM编码器能够处理图形和视频流，进行渲染和合成，然后将图像数据发送到显示设备以在屏幕上显示。
- 硬件加速： DRM编码器还可以利用硬件加速功能，以便更快地处理图形和视频内容。这有助于提高性能和效率，特别是对于高分辨率视频或图形内容的处理。
- 支持多种编解码标准： DRM编码器能够支持多种视频编解码标准，如H.264、H.265等，确保对不同格式的视频流进行正确的处理。
在Linux系统中，DRM编码器与图形驱动程序、图形处理单元（GPU）和显示设备等硬件密切相关。它允许Linux系统管理和控制图形硬件，处理视频流并在屏幕上显示图像，同时也与DRM系统中的访问控制和安全性机制集成，确保受保护内容的安全传输和显示。

{{< figure src="kms-display-encoder.svg" title="图8. encoder框架图" >}}

### connector
DRM connectors是用于管理和描述图形设备的连接器或端口。它们提供了对显示设备的连接和属性描述。在图形系统中，这些连接器可以代表诸如HDMI、DisplayPort、DVI等物理连接接口，而它们也可以描述内部连接，例如LCD panels。DRM connectors 在 Linux 中的作用包括：
- 显示设备连接描述： DRM connectors 提供了关于显示设备的物理连接信息，比如类型（HDMI、VGA、DisplayPort等）、连接状态以及支持的分辨率和刷新率等信息。
- 动态连接管理： 它们可以监测连接和断开事件。当显示设备插入或拔出时，DRM connectors 可以检测到这些变化，并通知系统相应的变化。这使系统能够动态调整图形配置以适应新的连接或断开状态。
- 多显示器支持： DRM connectors 允许系统管理多个显示器的连接和配置。这使得可以同时使用多个显示器或监视器，或者在不同的显示设备上显示不同的内容。
- 显示属性查询和配置： 通过 DRM connectors，可以查询连接器支持的分辨率、刷新率以及其他显示属性。这使系统可以调整和配置图形设备，以最佳方式显示图形内容。

总体来说，DRM connectors 在 Linux 中的作用是管理显示设备的物理连接、状态和属性，使系统能够动态适应不同的显示设备、支持多个显示器，并提供相应的配置信息，以便正确地显示图形内容。

{{< figure src="kms-display-connector.svg" title="图9. connector框架图" >}}

### framebuffer
framebuffer是一个重要概念。Framebuffers是用于存储屏幕上每个像素的颜色和其他相关信息的内存区域。在DRM中，framebuffer提供了一个抽象接口，允许用户空间程序访问和操作显存，以便渲染图形数据。其作用包括：
- 图形数据存储： Framebuffers提供一个区域，用于存储图形数据，包括每个像素的颜色、透明度等信息。这些数据构成了屏幕上显示的图像。
- 直接访问屏幕数据： 通过framebuffers，用户空间程序或操作系统内核能够直接访问和操作图形数据，而无需经过额外的复杂处理。
- 硬件抽象层： Framebuffers提供了一个硬件无关的抽象接口，这意味着不同的图形设备和硬件都可以通过相同的接口进行访问和操作。
- 显示图像： Framebuffers存储了最终用于在屏幕上显示的图像数据。图形数据在 framebuffer中进行组织和处理，然后通过显示控制器输出到屏幕。

在DRM 中，framebuffers通常与CRTC和显示控制器等组件一起工作。framebuffers提供的抽象层允许操作系统或应用程序以统一的方式对图形数据进行处理和管理，无论具体的硬件设备是什么样的。总之，framebuffers 在 Linux 的 DRM 中提供了一个接口和数据结构，用于存储、操作和管理图形数据，使其能够在屏幕上正确地显示图像。

{{< figure src="framebuffer-fields.svg" title="图10. framebuffer内存结构图" >}}

## drm调试工具
### drm_info
用于转储有关 DRM 设备信息的小实用程序
```shell
drm_info

Node: /dev/dri/card1
├───Driver: i915 (Intel Graphics) version 1.6.0 (20201103)
│   ├───DRM_CLIENT_CAP_STEREO_3D supported
│   ├───DRM_CLIENT_CAP_UNIVERSAL_PLANES supported
│   ├───DRM_CLIENT_CAP_ATOMIC supported
│   ├───DRM_CLIENT_CAP_ASPECT_RATIO supported
│   ├───DRM_CLIENT_CAP_WRITEBACK_CONNECTORS supported
│   ├───DRM_CAP_DUMB_BUFFER = 1
│   ├───DRM_CAP_VBLANK_HIGH_CRTC = 1
│   ├───DRM_CAP_DUMB_PREFERRED_DEPTH = 24
│   ├───DRM_CAP_DUMB_PREFER_SHADOW = 1
│   ├───DRM_CAP_PRIME = 3
│   ├───DRM_CAP_TIMESTAMP_MONOTONIC = 1
│   ├───DRM_CAP_ASYNC_PAGE_FLIP = 1
│   ├───DRM_CAP_CURSOR_WIDTH = 256
│   ├───DRM_CAP_CURSOR_HEIGHT = 256
│   ├───DRM_CAP_ADDFB2_MODIFIERS = 1
│   ├───DRM_CAP_PAGE_FLIP_TARGET = 0
│   ├───DRM_CAP_CRTC_IN_VBLANK_EVENT = 1
│   ├───DRM_CAP_SYNCOBJ = 1
│   └───DRM_CAP_SYNCOBJ_TIMELINE = 1
├───Device: PCI 8086:46aa Intel Corporation Alder Lake-UP4 GT2 [Iris Xe Graphics]
│   └───Available nodes: primary, render
├───Framebuffer size
│   ├───Width: [0, 16384]
│   └───Height: [0, 16384]
├───Connectors
│   ├───Connector 0
│   │   ├───Object ID: 236
│   │   ├───Type: eDP
│   │   ├───Status: connected
│   │   ├───Physical size: 290x180 mm
│   │   ├───Subpixel: unknown
│   │   ├───Encoders: {0}
│   │   ├───Modes
│   │   │   └───2880x1800@60.00 preferred driver phsync nvsync 
│   │   └───Properties
│   │       ├───"EDID" (immutable): blob = 266
│   │       ├───"DPMS": enum {On, Standby, Suspend, Off} = On
│   │       ├───"link-status": enum {Good, Bad} = Good
│   │       ├───"non-desktop" (immutable): range [0, 1] = 0
│   │       ├───"TILE" (immutable): blob = 0
│   │       ├───"CRTC_ID" (atomic): object CRTC = 80
│   │       ├───"scaling mode": enum {Full, Center, Full aspect} = Full aspect
│   │       ├───"panel orientation" (immutable): enum {Normal, Upside Down, Left Side Up, Right Side Up} = Normal
│   │       ├───"Broadcast RGB": enum {Automatic, Full, Limited 16:235} = Automatic
│   │       ├───"max bpc": range [6, 12] = 10
│   │       ├───"Colorspace": enum {Default, BT709_YCC, XVYCC_601, XVYCC_709, SYCC_601, opYCC_601, opRGB, BT2020_CYCC, BT2020_RGB, BT2020_YCC, DCI-P3_RGB_D65, RGB_WIDE_FIXED, RGB_WIDE_FLOAT, BT601_YCC} = Default
│   │       ├───"HDR_OUTPUT_METADATA": blob = 0
│   │       └───"vrr_capable" (immutable): range [0, 1] = 0
│   ├───Connector 1
│   │   ├───Object ID: 245
│   │   ├───Type: DisplayPort
│   │   ├───Status: disconnected
│   │   ├───Encoders: {1}
│   │   └───Properties
│   │       ├───"EDID" (immutable): blob = 0
│   │       ├───"DPMS": enum {On, Standby, Suspend, Off} = Off
│   │       ├───"link-status": enum {Good, Bad} = Good
│   │       ├───"non-desktop" (immutable): range [0, 1] = 0
│   │       ├───"TILE" (immutable): blob = 0
│   │       ├───"CRTC_ID" (atomic): object CRTC = 0
│   │       ├───"subconnector" (immutable): enum {Unknown, VGA, DVI-D, HDMI, DP, Wireless, Native} = Unknown
│   │       ├───"audio": enum {force-dvi, off, auto, on} = auto
│   │       ├───"Broadcast RGB": enum {Automatic, Full, Limited 16:235} = Automatic
│   │       ├───"max bpc": range [6, 12] = 12
│   │       ├───"Colorspace": enum {Default, BT709_YCC, XVYCC_601, XVYCC_709, SYCC_601, opYCC_601, opRGB, BT2020_CYCC, BT2020_RGB, BT2020_YCC, DCI-P3_RGB_D65, RGB_WIDE_FIXED, RGB_WIDE_FLOAT, BT601_YCC} = Default
│   │       ├───"HDR_OUTPUT_METADATA": blob = 0
│   │       ├───"vrr_capable" (immutable): range [0, 1] = 0
│   │       ├───"Content Protection": enum {Undesired, Desired, Enabled} = Undesired
│   │       └───"HDCP Content Type": enum {HDCP Type0, HDCP Type1} = HDCP Type0
│   └───Connector 2
│       ├───Object ID: 258
│       ├───Type: DisplayPort
│       ├───Status: disconnected
│       ├───Encoders: {6}
│       └───Properties
│           ├───"EDID" (immutable): blob = 0
│           ├───"DPMS": enum {On, Standby, Suspend, Off} = Off
│           ├───"link-status": enum {Good, Bad} = Good
│           ├───"non-desktop" (immutable): range [0, 1] = 0
│           ├───"TILE" (immutable): blob = 0
│           ├───"CRTC_ID" (atomic): object CRTC = 0
│           ├───"subconnector" (immutable): enum {Unknown, VGA, DVI-D, HDMI, DP, Wireless, Native} = Unknown
│           ├───"audio": enum {force-dvi, off, auto, on} = auto
│           ├───"Broadcast RGB": enum {Automatic, Full, Limited 16:235} = Automatic
│           ├───"max bpc": range [6, 12] = 12
│           ├───"Colorspace": enum {Default, BT709_YCC, XVYCC_601, XVYCC_709, SYCC_601, opYCC_601, opRGB, BT2020_CYCC, BT2020_RGB, BT2020_YCC, DCI-P3_RGB_D65, RGB_WIDE_FIXED, RGB_WIDE_FLOAT, BT601_YCC} = Default
│           ├───"HDR_OUTPUT_METADATA": blob = 0
│           ├───"vrr_capable" (immutable): range [0, 1] = 0
│           ├───"Content Protection": enum {Undesired, Desired, Enabled} = Undesired
│           └───"HDCP Content Type": enum {HDCP Type0, HDCP Type1} = HDCP Type0
├───Encoders
│   ├───Encoder 0
│   │   ├───Object ID: 235
│   │   ├───Type: TMDS
│   │   ├───CRTCS: {0, 1, 2, 3}
│   │   └───Clones: {0}
│   ├───Encoder 1
│   │   ├───Object ID: 244
│   │   ├───Type: TMDS
│   │   ├───CRTCS: {0, 1, 2, 3}
│   │   └───Clones: {1}

.........

├───CRTCs
│   ├───CRTC 0
│   │   ├───Object ID: 80
│   │   ├───Legacy info
│   │   │   ├───Mode: 2880x1800@60.00 preferred driver phsync nvsync 
│   │   │   └───Gamma size: 256
│   │   └───Properties
│   │       ├───"ACTIVE" (atomic): range [0, 1] = 1
│   │       ├───"MODE_ID" (atomic): blob = 271
│   │       │   └───2880x1800@60.00 preferred driver phsync nvsync 
│   │       ├───"OUT_FENCE_PTR" (atomic): range [0, UINT64_MAX] = 0
│   │       ├───"VRR_ENABLED": range [0, 1] = 0
│   │       ├───"SCALING_FILTER": enum {Default, Nearest Neighbor} = Default
│   │       ├───"DEGAMMA_LUT": blob = 0
│   │       ├───"DEGAMMA_LUT_SIZE" (immutable): range [0, UINT32_MAX] = 129
│   │       ├───"CTM": blob = 0
│   │       ├───"GAMMA_LUT": blob = 270
│   │       └───"GAMMA_LUT_SIZE" (immutable): range [0, UINT32_MAX] = 1024

........


└───Planes
    ├───Plane 0
    │   ├───Object ID: 31
    │   ├───CRTCs: {0}
    │   ├───Legacy info
    │   │   ├───FB ID: 237
    │   │   │   ├───Object ID: 237
    │   │   │   ├───Size: 2880x1800
    │   │   │   ├───Format: ARGB2101010 (0x30335241)
    │   │   │   ├───Modifier: I915_FORMAT_MOD_Y_TILED (0x100000000000002)
    │   │   │   └───Planes:
    │   │   │       └───Plane 0: offset = 0, pitch = 11520 bytes
    │   │   └───Formats:
    │   │       ├───C8 (0x20203843)
    │   │       ├───RGB565 (0x36314752)
    │   │       ├───XRGB8888 (0x34325258)
    │   │       ├───XBGR8888 (0x34324258)
    │   │       ├───ARGB8888 (0x34325241)
    │   │       ├───ABGR8888 (0x34324241)
    
    .....
```

### drm_monitor
用于监控 KMS 状态的 CLI 工具
```shell
drm_monitor -d /dev/dri/card1
CRTC 80: seq=30095325875 ns=3076787603594 delta_ns=16668012 Hz=59.995157
CRTC 131: seq=0 ns=0 delta_ns=0 Hz=0.000000
CRTC 182: seq=0 ns=0 delta_ns=0 Hz=0.000000
CRTC 233: seq=0 ns=0 delta_ns=0 Hz=0.000000
```

## 参考资料
- [Kernel Mode Setting](https://docs.kernel.org/gpu/drm-kms.html)
- [brezillon-drm-kms](https://bootlin.com/pub/conferences/2014/elce/brezillon-drm-kms/)
- [drm_info](https://gitlab.freedesktop.org/emersion/drm_info)
- [drm_monitor](https://github.com/emersion/drm_monitor)