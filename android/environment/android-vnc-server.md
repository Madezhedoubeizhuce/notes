# 安装ndk

1. 下载ndk，最好不要下载新版本，此处下载r14b版本
2. 配置环境变量ANDROID_NDK

``` shell
export ANDROID_NDK=/home/despair/softwares/ndk/android-ndk-r14b
export PATH=$ANDROID_NDK:$PATH
```

# 编译libjpeg

1. 下载地址<http://www.ijg.org/>
2. 配置ndk的gcc路径

```shell
export SYSROOT=$ANDROID_NDK/platforms/android-16/arch-arm
export CC="$ANDROID_NDK/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/bin/arm-linux-androideabi-gcc-4.9 --sysroot=$SYSROOT"
```

3. 解压libjpeg，并cd到jpeg目录，进行configure，手动指明prefix目录和exec-prefix目录

``` shell
./configure --prefix=/home/despair/source_code/jpeg-9c/prefix --exec-prefix=/home/despair/source_code/jpeg-9c/exec-prefix --host=arm
```

4. 编译，完成后，在configure中指定的prefix和exec-prefix的文件夹中可以找到编译后得到的头文件和库文件。

``` shell
make
make install
```

# 编译android-vnc-server

1. 下载代码：谷歌官方库（<https://code.google.com/archive/p/android-vnc-server/>），github镜像（https://github.com/maurossi/android-vnc-server.git），因为github能够直接下载，推荐直接用github的连接

2. 在源代码目录下，新建jni目录，将下载好的源代码都剪切至该目录
3. 在jni目录，新建Application.mk文件，添加`APP_ABI := aremabi armeabi-v7a`，保存文件退出

``` makefile
APP_ABI := aremabi armeabi-v7a
APP_PLATFORM := android-16
```

2. 修改LibVNCServer-0.9.7/libvncserver/main.c文件中的第245行，将`sprintf(stderr,buf);` 修改为`sprintf(stderr,"%s",buf);` 不然编译的时候会出错。
3. 打开Android.mk,发现它需要使用第三方的库文件libz.so和libjpeg.a，libz.so在ndk中已经支持，我们需要编译libjpeg，参考前一节
4. 在jni目录项新建一个include文件夹，将libjpeg的头文件拷贝至该目录(jconfig.h, jerror.h, jmorecfg.h, jpeglib.h)
5. 将编译的libjpeg.a静态库拷贝至jni目录
6. 修改Android.mk文件

```makefile
# 添加jpeg的头文件
LOCAL_C_INCLUDES := \
	$(LOCAL_PATH) \
	$(LOCAL_PATH)/LibVNCServer-0.9.7/libvncserver \
	$(LOCAL_PATH)/LibVNCServer-0.9.7 \
	$(LOCAL_PATH)/include \
	external/zlib \
	external/jpeg
	
# 改用LOCAL_LDLIBS指定库
#LOCAL_SHARED_LIBRARIES := libz
#LOCAL_STATIC_LIBRARIES := libjpeg
LOCAL_LDLIBS := $(LOCAL_PATH)/libjpeg.a -lz

# 添加这两行，解决出现error: only position independent executables (PIE) are supported.
LOCAL_CFLAGS += -pie -fPIE
LOCAL_LDFLAGS += -pie -fPIE
```

9. 注释掉fbvncserver.c第246行的`KEY_SOFT1, KEY_SOFT2,`第267行、269行、270行和283行。

```c
static int keysym2scancode(rfbBool down, rfbKeySym key, rfbClientPtr cl)
{
    int scancode = 0;

    int code = (int)key;
    if (code>='0' && code<='9') {
        scancode = (code & 0xF) - 1;
        if (scancode<0) scancode += 10;
        scancode += KEY_1;
    } else if (code>=0xFF50 && code<=0xFF58) {
        static const uint16_t map[] =
             {  KEY_HOME, KEY_LEFT, KEY_UP, KEY_RIGHT, KEY_DOWN,
                /*KEY_SOFT1, KEY_SOFT2,*/ KEY_END, 0 };
        scancode = map[code & 0xF];
    } else if (code>=0xFFE1 && code<=0xFFEE) {
        static const uint16_t map[] =
             {  KEY_LEFTSHIFT, KEY_LEFTSHIFT,
                KEY_COMPOSE, KEY_COMPOSE,
                KEY_LEFTSHIFT, KEY_LEFTSHIFT,
                0,0,
                KEY_LEFTALT, KEY_RIGHTALT,
                0, 0, 0, 0 };
        scancode = map[code & 0xF];
    } else if ((code>='A' && code<='Z') || (code>='a' && code<='z')) {
        static const uint16_t map[] = {
                KEY_A, KEY_B, KEY_C, KEY_D, KEY_E,
                KEY_F, KEY_G, KEY_H, KEY_I, KEY_J,
                KEY_K, KEY_L, KEY_M, KEY_N, KEY_O,
                KEY_P, KEY_Q, KEY_R, KEY_S, KEY_T,
                KEY_U, KEY_V, KEY_W, KEY_X, KEY_Y, KEY_Z };
        scancode = map[(code & 0x5F) - 'A'];
    } else {
        switch (code) {
            //case 0x0003:    scancode = KEY_CENTER;      break;
            case 0x0020:    scancode = KEY_SPACE;       break;
            //case 0x0023:    scancode = KEY_SHARP;       break;
            //case 0x0033:    scancode = KEY_SHARP;       break;
            case 0x002C:    scancode = KEY_COMMA;       break;
            case 0x003C:    scancode = KEY_COMMA;       break;
            case 0x002E:    scancode = KEY_DOT;         break;
            case 0x003E:    scancode = KEY_DOT;         break;
            case 0x002F:    scancode = KEY_SLASH;       break;
            case 0x003F:    scancode = KEY_SLASH;       break;
            case 0x0032:    scancode = KEY_EMAIL;       break;
            case 0x0040:    scancode = KEY_EMAIL;       break;
            case 0xFF08:    scancode = KEY_BACKSPACE;   break;
            case 0xFF1B:    scancode = KEY_BACK;        break;
            case 0xFF09:    scancode = KEY_TAB;         break;
            case 0xFF0D:    scancode = KEY_ENTER;       break;
            //case 0x002A:    scancode = KEY_STAR;        break;
            case 0xFFBE:    scancode = KEY_F1;        break; // F1
            case 0xFFBF:    scancode = KEY_F2;         break; // F2
            case 0xFFC0:    scancode = KEY_F3;        break; // F3
            case 0xFFC5:    scancode = KEY_F4;       break; // F8
            case 0xFFC8:    rfbShutdownServer(cl->screen,TRUE);       break; // F11            
        }
    }
```

10. 使用ndk-build

```shell
ndk-build APP_ABI=armeabi-v7a,armeabi
```

