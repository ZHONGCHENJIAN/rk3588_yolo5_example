# 本功能基于以下硬件实现
+ 开发板：香橙派RK3588
+ 摄像头：OV13855
+ LCD：5寸 mipi lcd （1080*1920像素）

# 摄像头采集
官方代码是通过opencv打开摄像头的，但是我在香橙派RK3588上一直无法打开，所以直接采用v4l2库配置摄像头进行采集。相关功能参考正点原子的代码实现。

## v4L2设备初始化
```c
static int v4l2_dev_init(const char *device)
{
    struct v4l2_capability cap = {0};

    /* 打开摄像头 */
    v4l2_fd = open(device, O_RDWR);
    if (0 > v4l2_fd) {
        fprintf(stderr, "open error: %s: %s\n", device, strerror(errno));
        return -1;
    }

    /* 查询设备功能 */
    ioctl(v4l2_fd, VIDIOC_QUERYCAP, &cap);

    /* 判断是否是视频采集设备 */
    if (!(V4L2_CAP_VIDEO_CAPTURE_MPLANE & cap.capabilities)) {
        fprintf(stderr, "Error: %s: No capture video device!\n", device);
        close(v4l2_fd);
        return -1;
    }

    return 0;
}
```

注:这里摄像头驱动的视频采集能力是V4L2_CAP_VIDEO_CAPTURE_MPLANE。

## 获取及显示摄像头像素格式
```c
static void v4l2_enum_formats(void)
{
    struct v4l2_fmtdesc fmtdesc = {0};

    /* 枚举摄像头所支持的所有像素格式以及描述信息 */
    fmtdesc.index = 0;
    fmtdesc.type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;
    while (0 == ioctl(v4l2_fd, VIDIOC_ENUM_FMT, &fmtdesc)) {

        // 将枚举出来的格式以及描述信息存放在数组中
        cam_fmts[fmtdesc.index].pixelformat = fmtdesc.pixelformat;
        strcpy(cam_fmts[fmtdesc.index].description, fmtdesc.description);
        fmtdesc.index++;
    }
}
```

```c
static void v4l2_print_formats(void)
{
    struct v4l2_frmsizeenum frmsize = {0};
    struct v4l2_frmivalenum frmival = {0};
    int i;

    frmsize.type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;
    frmival.type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;
    for (i = 0; cam_fmts[i].pixelformat; i++) {

        printf("format<0x%x>, description<%s>\n", cam_fmts[i].pixelformat,
                    cam_fmts[i].description);

        /* 枚举出摄像头所支持的所有视频采集分辨率 */
        frmsize.index = 0;
        frmsize.pixel_format = cam_fmts[i].pixelformat;
        frmival.pixel_format = cam_fmts[i].pixelformat;
        while (0 == ioctl(v4l2_fd, VIDIOC_ENUM_FRAMESIZES, &frmsize)) {

            printf("size<%d*%d> ",
                    frmsize.discrete.width,
                    frmsize.discrete.height);
            frmsize.index++;

            /* 获取摄像头视频采集帧率 */
            frmival.index = 0;
            frmival.width = frmsize.discrete.width;
            frmival.height = frmsize.discrete.height;
            while (0 == ioctl(v4l2_fd, VIDIOC_ENUM_FRAMEINTERVALS, &frmival)) {
                printf("<%dfps>", frmival.discrete.denominator /
                        frmival.discrete.numerator);
                frmival.index++;
            }
            printf("\n");
        }
        printf("\n");
    }
}
```

## 设置帧格式、帧率等
```c
static int v4l2_set_format(void)
{
    struct v4l2_format fmt = {0};
    struct v4l2_streamparm streamparm = {0};

    /* 设置帧格式 */
    fmt.type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;//type类型
    fmt.fmt.pix.width = 640;  //视频帧宽度
    fmt.fmt.pix.height = 480;//视频帧高度
    fmt.fmt.pix.pixelformat = V4L2_PIX_FMT_NV12;  //像素格式
    if (0 > ioctl(v4l2_fd, VIDIOC_S_FMT, &fmt)) {
        fprintf(stderr, "ioctl error: VIDIOC_S_FMT: %s\n", strerror(errno));
        return -1;
    }

    /*** 判断是否已经设置为我们要求的RGB565像素格式
    如果没有设置成功表示该设备不支持RGB565像素格式 */
    if (V4L2_PIX_FMT_NV12 != fmt.fmt.pix.pixelformat) {
        fprintf(stderr, "Error: the device does not support V4L2_PIX_FMT_NV12 format!\n");
        return -1;
    }

    fmt.type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;
	// 显示当前的图层格式
    if(ioctl(v4l2_fd, VIDIOC_G_FMT, &fmt) == -1) {
        printf("Unable to get format\n");
        return -1;
    }
    printf("fmt.type:\t\t%d\n",fmt.type);
    printf("pix.pixelformat:\t%c%c%c%c\n",fmt.fmt.pix_mp.pixelformat & 0xFF, (fmt.fmt.pix_mp.pixelformat >> 8) & 0xFF,(fmt.fmt.pix_mp.pixelformat >> 16) & 0xFF, (fmt.fmt.pix_mp.pixelformat >> 24) & 0xFF);
    printf("pix.height:\t\t%d\n",fmt.fmt.pix_mp.height);
    printf("pix.width:\t\t%d\n",fmt.fmt.pix_mp.width);
    printf("pix.field:\t\t%d\n",fmt.fmt.pix_mp.field);
    
    frm_width = fmt.fmt.pix.width;  //获取实际的帧宽度
    frm_height = fmt.fmt.pix.height;//获取实际的帧高度
    printf("视频帧大小<%d * %d>\n", frm_width, frm_height);

    /* 获取streamparm */
    streamparm.type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;
    ioctl(v4l2_fd, VIDIOC_G_PARM, &streamparm);

    /** 判断是否支持帧率设置 **/
    if (V4L2_CAP_TIMEPERFRAME & streamparm.parm.capture.capability) {
        streamparm.parm.capture.timeperframe.numerator = 1;
        streamparm.parm.capture.timeperframe.denominator = 60;//30fps
        if (0 > ioctl(v4l2_fd, VIDIOC_S_PARM, &streamparm)) {
            fprintf(stderr, "ioctl error: VIDIOC_S_PARM: %s\n", strerror(errno));
            return -1;
        }
    }

    return 0;
}
```

官方给的YOLO5例程的输入图片格式是640*480尺寸，因此我们这里直接设置摄像头采集帧尺寸为640*480，方便后续喂图。另外图片编码是NV12，后续需要转为RGB555。

## 初始化帧缓冲buffer
```c
static int v4l2_init_buffer(void)
{
    struct v4l2_requestbuffers reqbuf = {0};
    struct v4l2_buffer buf = {0};
 

    /* 申请帧缓冲 */
    reqbuf.count = FRAMEBUFFER_COUNT;       //帧缓冲的数量
    reqbuf.type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;
    reqbuf.memory = V4L2_MEMORY_MMAP;
    if (0 > ioctl(v4l2_fd, VIDIOC_REQBUFS, &reqbuf)) {
        fprintf(stderr, "ioctl error: VIDIOC_REQBUFS: %s\n", strerror(errno));
        return -1;
    }

    /* 建立内存映射 */
    for (int i = 0; i < FRAMEBUFFER_COUNT; i++) {
        struct v4l2_plane planes[FMT_NUM_PLANES];
        memset(&buf, 0, sizeof(buf));

        buf.type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;
        buf.memory = V4L2_MEMORY_MMAP;
        buf.length = FMT_NUM_PLANES;
        buf.m.planes = planes;
        buf.index = i;
        if (ioctl(v4l2_fd, VIDIOC_QUERYBUF, &buf) < 0) {
			perror("VIDIOC_QUERYBUF");
			return -1;
		}
        for (int j = 0; j < FMT_NUM_PLANES; j++) {
            buf_infos[i].length[j] = buf.m.planes[j].length;

            buf_infos[i].start[j] = mmap(NULL, buf.m.planes[j].length,
                PROT_READ | PROT_WRITE, MAP_SHARED,
                v4l2_fd, buf.m.planes[j].m.mem_offset);
            if (MAP_FAILED == buf_infos[i].start[j]) {
                perror("v4l mmap error");
                return -1;
            }
        } 
    }

    /* 入队 */
    for (buf.index = 0; buf.index < FRAMEBUFFER_COUNT; buf.index++) {

        if (0 > ioctl(v4l2_fd, VIDIOC_QBUF, &buf)) {
            fprintf(stderr, "ioctl error: VIDIOC_QBUF: %s\n", strerror(errno));
            return -1;
        }
    }
    
    return 0;
}
```

## 开始采集数据
```c
static int v4l2_stream_on(void)
{
    /* 打开摄像头、摄像头开始采集数据 */
    enum v4l2_buf_type type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;

    if (0 > ioctl(v4l2_fd, VIDIOC_STREAMON, &type)) {
        fprintf(stderr, "ioctl error: VIDIOC_STREAMON: %s\n", strerror(errno));
        return -1;
    }

    return 0;
}
```

## 实时帧读取处理（包含yolo5相关代码）
```c
static int v4l2_read_data(void)
{
    struct v4l2_buffer buf = {0};
    struct v4l2_plane planes[FMT_NUM_PLANES];

    unsigned char* nv12_data = (unsigned char*)malloc(frm_width * frm_height + (frm_width * frm_height / 2));
    unsigned char* rgba_data = (unsigned char*)malloc(frm_width * frm_height * 4);
    char* lcd_data1 = (char*)malloc(640 * 480 * 4);
    char* lcd_data = (char*)malloc(1080 * 1920 * 4);
    unsigned char* screen_base_temp_data = (unsigned char*)malloc(width * height * 4);
    if (!nv12_data || !rgba_data) {
        perror("Error allocating memory for image buffers");
        return -1;
    }

    buf.type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;
    buf.memory = V4L2_MEMORY_MMAP;
    buf.length = FMT_NUM_PLANES;
    buf.m.planes = planes;
    
    const char *model_path = "../model/yolov5.rknn";
    int ret;
    rknn_app_context_t rknn_app_ctx;
    memset(&rknn_app_ctx, 0, sizeof(rknn_app_context_t));

    init_post_process();

    ret = init_yolov5_model(model_path, &rknn_app_ctx);
    if (ret != 0)
    {
        printf("init_yolov5_model fail! ret=%d model_path=%s\n", ret, model_path);
    }

    image_buffer_t src_image;
    memset(&src_image, 0, sizeof(image_buffer_t));
    object_detect_result_list od_results;
    cv::Mat src_frame;
    for ( ; ; ) {
        for(buf.index = 0; buf.index < FRAMEBUFFER_COUNT; buf.index++) {
            // printf("Attempting to dequeue buffer with index: %d\n", buf.index);
            if (ioctl(v4l2_fd, VIDIOC_DQBUF, &buf) < 0) {
				perror("failed to dequeue\n");
				return -1;
			}
           
            memcpy(nv12_data, buf_infos[buf.index].start[0], buf_infos[buf.index].length[0]);
            
            NV12_to_RGBA(nv12_data, rgba_data, frm_width, frm_height);
            
            cv::Mat rgba_image(frm_height, frm_width, CV_8UC4, rgba_data);
            cv::cvtColor(rgba_image, rgba_image, cv::COLOR_BGRA2BGR);
            cv::rotate(rgba_image, src_frame, cv::ROTATE_90_COUNTERCLOCKWISE);
            src_image.height = src_frame.rows;
            src_image.width = src_frame.cols;
            src_image.width_stride = src_frame.step[0];
            src_image.virt_addr = src_frame.data;
            src_image.format = IMAGE_FORMAT_RGB888;
            src_image.size = src_frame.total() * src_frame.elemSize();

            image_buffer_t dst_img;
            letterbox_t letter_box;
            rknn_input inputs[rknn_app_ctx.io_num.n_input];
            rknn_output outputs[rknn_app_ctx.io_num.n_output];
            const float nms_threshold = NMS_THRESH;      // Default NMS threshold
            const float box_conf_threshold = BOX_THRESH; // Default box threshold
            int bg_color = 114;

            memset(&od_results, 0x00, sizeof(od_results));
            memset(&letter_box, 0, sizeof(letterbox_t));
            memset(&dst_img, 0, sizeof(image_buffer_t));
            memset(inputs, 0, sizeof(inputs));
            memset(outputs, 0, sizeof(outputs));

            // Pre Process
            dst_img.width = rknn_app_ctx.model_width;
            dst_img.height = rknn_app_ctx.model_height;
            dst_img.format = IMAGE_FORMAT_RGB888;
            dst_img.size = get_image_size(&dst_img);
            dst_img.virt_addr = (unsigned char *)malloc(dst_img.size);
            if (dst_img.virt_addr == NULL)
            {
                printf("malloc buffer size:%d fail!\n", dst_img.size);
                return -1;
            }

            // letterbox
            ret = convert_image_with_letterbox(&src_image, &dst_img, &letter_box, bg_color);
            if (ret < 0)
            {
                printf("convert_image_with_letterbox fail! ret=%d\n", ret);
                return -1;
            }

            // Set Input Data
            inputs[0].index = 0;
            inputs[0].type = RKNN_TENSOR_UINT8;
            inputs[0].fmt = RKNN_TENSOR_NHWC;
            inputs[0].size = rknn_app_ctx.model_width * rknn_app_ctx.model_height * rknn_app_ctx.model_channel;
            inputs[0].buf = dst_img.virt_addr;

            ret = rknn_inputs_set(rknn_app_ctx.rknn_ctx, rknn_app_ctx.io_num.n_input, inputs);
            if (ret < 0)
            {
                printf("rknn_input_set fail! ret=%d\n", ret);
                return -1;
            }
            free(dst_img.virt_addr);
            // Run
            printf("rknn_run\n");
            ret = rknn_run(rknn_app_ctx.rknn_ctx, nullptr);
            if (ret < 0)
            {
                printf("rknn_run fail! ret=%d\n", ret);
                return -1;
            }

            // Get Output
            memset(outputs, 0, sizeof(outputs));
            for (int i = 0; i < rknn_app_ctx.io_num.n_output; i++)
            {
                outputs[i].index = i;
                outputs[i].want_float = (!rknn_app_ctx.is_quant);
            }
            ret = rknn_outputs_get(rknn_app_ctx.rknn_ctx, rknn_app_ctx.io_num.n_output, outputs, NULL);
            // Post Process
            post_process(&rknn_app_ctx, outputs, &letter_box, box_conf_threshold, nms_threshold, &od_results);

            // Remeber to release rknn output
            rknn_outputs_release(rknn_app_ctx.rknn_ctx, rknn_app_ctx.io_num.n_output, outputs);
            // 画框
            char text[256];
            printf("<<<<<<<<<<<od_results.count :%d<<<<<<<<<<<<<",od_results.count);
            for (int i = 0; i < od_results.count; i++)
            {
                object_detect_result *det_result = &(od_results.results[i]);
                printf("%s @ (%d %d %d %d) %.3f\n", coco_cls_to_name(det_result->cls_id),
                    det_result->box.left, det_result->box.top,
                    det_result->box.right, det_result->box.bottom,
                    det_result->prop);
                int x1 = det_result->box.left;
                int y1 = det_result->box.top;
                int x2 = det_result->box.right;
                int y2 = det_result->box.bottom;

                draw_rectangle(&src_image, x1, y1, x2 - x1, y2 - y1, COLOR_BLUE, 3);

                sprintf(text, "%s %.1f%%", coco_cls_to_name(det_result->cls_id), det_result->prop * 100);
                draw_text(&src_image, text, x1, y1 - 20, COLOR_GREEN, 10);
            }
            std::memcpy(lcd_data1, src_image.virt_addr, src_image.size);
            rga_resize(lcd_data1, lcd_data, 480, 640, width, height, RK_FORMAT_BGR_888, RK_FORMAT_RGBA_8888);
            memcpy(screen_base, lcd_data, width*height*4);

            // 数据处理完之后、再入队、往复
            ioctl(v4l2_fd, VIDIOC_QBUF, &buf);
        }
    }
    free(nv12_data);
    free(rgba_data);
    free(lcd_data);
    free(lcd_data1);
}
```

# lcd
## 初始化
```c
static int fb_dev_init(void)
{
    struct fb_var_screeninfo fb_var = {0};
    struct fb_fix_screeninfo fb_fix = {0};
    unsigned long screen_size;

    /* 打开framebuffer设备 */
    fb_fd = open(FB_DEV, O_RDWR);
    if (0 > fb_fd) {
        fprintf(stderr, "open error: %s: %s\n", FB_DEV, strerror(errno));
        return -1;
    }

    /* 获取framebuffer设备信息 */
    ioctl(fb_fd, FBIOGET_VSCREENINFO, &fb_var);
    ioctl(fb_fd, FBIOGET_FSCREENINFO, &fb_fix);

    screen_size = fb_fix.line_length * fb_var.yres;
    width = fb_var.xres;
    height = fb_var.yres;
    line_length = fb_fix.line_length / (fb_var.bits_per_pixel / 8);

    printf("screen width:%d height:%d\n",width, height);
    /* 内存映射 */
    screen_base = mmap(NULL, screen_size, PROT_READ | PROT_WRITE, MAP_SHARED, fb_fd, 0);
    if (MAP_FAILED == (void *)screen_base) {
        perror("screen mmap error");
        close(fb_fd);
        return -1;
    }

    /* LCD背景刷白 */
    memset(screen_base, 0x00, screen_size);
    return 0;
}
```

+ 打开lcd设备，一般是/dev/fb0，
+ 获取lcd的width和height
+ 内存映射，从内核空间映射到用户空间，后续只需要将图片数据按RGBA格式写到screen_base指针处，LCD立马会显示出相应图片

## 图片显示
```c
 memcpy(screen_base, lcd_data, width*height*4);
```

# yolo5
## NV12格式转RGB
yolo5模型输入的图片格式是RGB，因此通过官方提供的rga库(硬件图片处理)进行图片格式转换

```c
rga_cvcolor(nv12_data, rgb_data, frm_width, frm_height, frm_width, frm_height, RK_FORMAT_YCbCr_420_SP, RK_FORMAT_RGB_888);
int rga_cvcolor(char *src_buf, char*dst_buf, int src_width, int src_height, int dst_width, int dst_height, int src_format,  int dst_format)
{
    int ret = 0;
    int src_buf_size, dst_buf_size;

    rga_buffer_t src_img, dst_img;
    rga_buffer_handle_t src_handle, dst_handle;

    memset(&src_img, 0, sizeof(src_img));
    memset(&dst_img, 0, sizeof(dst_img));

    src_buf_size = src_width * src_height * get_bpp_from_format(src_format);
    dst_buf_size = dst_width * dst_height * get_bpp_from_format(dst_format);

    memset(dst_buf, 0x80, dst_buf_size);

    src_handle = importbuffer_virtualaddr(src_buf, src_buf_size);
    dst_handle = importbuffer_virtualaddr(dst_buf, dst_buf_size);
    if (src_handle == 0 || dst_handle == 0) {
        printf("importbuffer failed!\n");
        goto release_buffer;
    }

    src_img = wrapbuffer_handle(src_handle, src_width, src_height, src_format);
    dst_img = wrapbuffer_handle(dst_handle, dst_width, dst_height, dst_format);

     ret = imcheck(src_img, dst_img, {}, {});
    if (IM_STATUS_NOERROR != ret) {
        printf("%d, check error! %s", __LINE__, imStrError((IM_STATUS)ret));
        return -1;
    }

    ret = imcvtcolor(src_img, dst_img, src_format, dst_format);
    if (ret == IM_STATUS_SUCCESS) {
        printf("running success!\n");
    } else {
        printf("running failed, %s\n", imStrError((IM_STATUS)ret));
        goto release_buffer;
    }

release_buffer:
    if (src_handle)
        releasebuffer_handle(src_handle);
    if (dst_handle)
        releasebuffer_handle(dst_handle);
    return ret;
}
```

## yolo5相关处理
```c
cv::Mat rgb_image(frm_height, frm_width, CV_8UC3, rgb_data);
cv::rotate(rgb_image, src_frame, cv::ROTATE_90_COUNTERCLOCKWISE);
src_image.height = src_frame.rows;
src_image.width = src_frame.cols;
src_image.width_stride = src_frame.step[0];
src_image.virt_addr = src_frame.data;
src_image.format = IMAGE_FORMAT_RGB888;
src_image.size = src_frame.total() * src_frame.elemSize();

image_buffer_t dst_img;
letterbox_t letter_box;
rknn_input inputs[rknn_app_ctx.io_num.n_input];
rknn_output outputs[rknn_app_ctx.io_num.n_output];
const float nms_threshold = NMS_THRESH;      // Default NMS threshold
const float box_conf_threshold = BOX_THRESH; // Default box threshold
int bg_color = 114;

memset(&od_results, 0x00, sizeof(od_results));
memset(&letter_box, 0, sizeof(letterbox_t));
memset(&dst_img, 0, sizeof(image_buffer_t));
memset(inputs, 0, sizeof(inputs));
memset(outputs, 0, sizeof(outputs));

// Pre Process
dst_img.width = rknn_app_ctx.model_width;
dst_img.height = rknn_app_ctx.model_height;
dst_img.format = IMAGE_FORMAT_RGB888;
dst_img.size = get_image_size(&dst_img);
dst_img.virt_addr = (unsigned char *)malloc(dst_img.size);
if (dst_img.virt_addr == NULL)
{
    printf("malloc buffer size:%d fail!\n", dst_img.size);
    return -1;
}

// letterbox
ret = convert_image_with_letterbox(&src_image, &dst_img, &letter_box, bg_color);
if (ret < 0)
{
    printf("convert_image_with_letterbox fail! ret=%d\n", ret);
    return -1;
}

// Set Input Data
inputs[0].index = 0;
inputs[0].type = RKNN_TENSOR_UINT8;
inputs[0].fmt = RKNN_TENSOR_NHWC;
inputs[0].size = rknn_app_ctx.model_width * rknn_app_ctx.model_height * rknn_app_ctx.model_channel;
inputs[0].buf = dst_img.virt_addr;

ret = rknn_inputs_set(rknn_app_ctx.rknn_ctx, rknn_app_ctx.io_num.n_input, inputs);
if (ret < 0)
{
    printf("rknn_input_set fail! ret=%d\n", ret);
    return -1;
}
free(dst_img.virt_addr);
// Run
printf("rknn_run\n");
ret = rknn_run(rknn_app_ctx.rknn_ctx, nullptr);
if (ret < 0)
{
    printf("rknn_run fail! ret=%d\n", ret);
    return -1;
}

// Get Output
memset(outputs, 0, sizeof(outputs));
for (int i = 0; i < rknn_app_ctx.io_num.n_output; i++)
{
    outputs[i].index = i;
    outputs[i].want_float = (!rknn_app_ctx.is_quant);
}
ret = rknn_outputs_get(rknn_app_ctx.rknn_ctx, rknn_app_ctx.io_num.n_output, outputs, NULL);
// Post Process
post_process(&rknn_app_ctx, outputs, &letter_box, box_conf_threshold, nms_threshold, &od_results);

// Remeber to release rknn output
rknn_outputs_release(rknn_app_ctx.rknn_ctx, rknn_app_ctx.io_num.n_output, outputs);
// 画框
char text[256];
printf("<<<<<<<<<<<od_results.count :%d<<<<<<<<<<<<<",od_results.count);
for (int i = 0; i < od_results.count; i++)
{
    object_detect_result *det_result = &(od_results.results[i]);
    printf("%s @ (%d %d %d %d) %.3f\n", coco_cls_to_name(det_result->cls_id),
        det_result->box.left, det_result->box.top,
        det_result->box.right, det_result->box.bottom,
        det_result->prop);
    int x1 = det_result->box.left;
    int y1 = det_result->box.top;
    int x2 = det_result->box.right;
    int y2 = det_result->box.bottom;

    draw_rectangle(&src_image, x1, y1, x2 - x1, y2 - y1, COLOR_BLUE, 3);

    sprintf(text, "%s %.1f%%", coco_cls_to_name(det_result->cls_id), det_result->prop * 100);
    draw_text(&src_image, text, x1, y1 - 20, COLOR_GREEN, 10);
}
```

```c
std::memcpy(lcd_data1, src_image.virt_addr, src_image.size);
rga_resize(lcd_data1, lcd_data, 480, 640, width, height, RK_FORMAT_BGR_888, RK_FORMAT_RGBA_8888);
memcpy(screen_base, lcd_data, width*height*4);
```

yolo5模型输出是480*640尺寸，因此需要通过rga将其缩放到1080*1920，正好填充LCD显示。

# 工程文件
├── 3rdparty

├── build

├── CMakeLists.txt

├── main.cc

├── model

├── opencv_3.4.15_aarch64

├── opencv_3.4.15_aarch64.tar

├── postprocess.cc

├── postprocess.h

├── rknpu2

├── utils

└── yolov5.h

官方yolo5工程的opencv库不全，因此交叉编译了一个opencv_3.4.15_aarch64

工程已提交至github：

