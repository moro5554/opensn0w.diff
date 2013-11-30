opensn0w.diff
=============
diff -urN opensn0w_origin/include/libirecovery.h opensn0w/include/libirecovery.h
--- opensn0w_origin/include/libirecovery.h	2013-09-28 00:27:01.000000000 +0900
+++ opensn0w/include/libirecovery.h	2013-09-28 00:15:42.000000000 +0900
@@ -36,7 +36,7 @@
 #undef interface
 #endif
 
-#ifdef __APPLE__
+#ifndef __APPLE__
 #include <CoreFoundation/CoreFoundation.h>
 #include <IOKit/IOKitLib.h>
 #include <IOKit/usb/IOUSBLib.h>
@@ -153,7 +153,7 @@
 		int alt_interface;
 		unsigned short mode;
 		char serial[256];
-#ifdef __APPLE__
+#ifndef __APPLE__
         __apple_usb_context handle;
 #elif !defined(WIN32)
 		libusb_device_handle *handle;
diff -urN opensn0w_origin/include/libusb-1.0/libusbi.h opensn0w/include/libusb-1.0/libusbi.h
--- opensn0w_origin/include/libusb-1.0/libusbi.h	2013-09-28 00:27:01.000000000 +0900
+++ opensn0w/include/libusb-1.0/libusbi.h	2013-09-28 00:39:26.000000000 +0900
@@ -23,6 +23,8 @@
 
 #include <config.h>
 
+#include <poll.h>
+#include <pthread.h>
 #include <stddef.h>
 #include <stdint.h>
 #include <time.h>
@@ -198,6 +200,9 @@
 #include <os/poll_windows.h>
 #endif
 
+#define usbi_mutex_t pthread_mutex_t
+#define usbi_cond_t pthread_cond_t
+
 extern struct libusb_context *usbi_default_context;
 
 struct libusb_context {
diff -urN opensn0w_origin/include/libusb-1.0/os/libusbi.h opensn0w/include/libusb-1.0/os/libusbi.h
--- opensn0w_origin/include/libusb-1.0/os/libusbi.h	2013-09-28 00:27:01.000000000 +0900
+++ opensn0w/include/libusb-1.0/os/libusbi.h	2013-09-28 00:36:49.000000000 +0900
@@ -23,6 +23,8 @@
 
 #include <config.h>
 
+#include <poll.h>
+#include <pthread.h>
 #include <stddef.h>
 #include <stdint.h>
 #include <time.h>
@@ -198,6 +200,9 @@
 #include <os/poll_windows.h>
 #endif
 
+#define usbi_mutex_t pthread_mutex_t
+#define usbi_cond_t pthread_cond_t
+
 extern struct libusb_context *usbi_default_context;
 
 struct libusb_context {
diff -urN opensn0w_origin/libsn0wcore/libirecovery.c opensn0w/libsn0wcore/libirecovery.c
--- opensn0w_origin/libsn0wcore/libirecovery.c	2013-09-28 00:27:01.000000000 +0900
+++ opensn0w/libsn0wcore/libirecovery.c	2013-09-28 00:20:54.000000000 +0900
@@ -25,6 +25,9 @@
 #include "dprint.h"
 #if !defined(WIN32)
 #include <libusb-1.0/libusb.h>
+#ifdef __APPLE__
+#include <libusb-1.0/os/darwin_usb.h>
+#endif
 #else
 #ifndef WIN32_LEAN_AND_MEAN
 #define WIN32_LEAN_AND_MEAN
@@ -43,7 +46,7 @@
 #endif
 
 static int libirecovery_debug = 0;
-const int DeviceVersion = 320;
+//const int DeviceVersion = 320;
 
 #if !defined(WIN32)
 static libusb_context *libirecovery_context = NULL;
@@ -298,7 +301,7 @@
 
 int check_context(irecv_client_t client)
 {
-#ifdef __APPLE__
+#ifndef __APPLE__
 	if (client == NULL || client->handle.device == NULL) {
 #else
     if (client == NULL || client->handle == NULL) {
@@ -311,7 +314,7 @@
 
 void irecv_init()
 {
-#ifdef __APPLE__
+#ifndef __APPLE__
     // kIOMasterPortDefault is the MasterPort.
 #elif !defined(WIN32)
 	libusb_init(&libirecovery_context);
@@ -320,7 +323,7 @@
 
 void irecv_exit()
 {
-#ifdef __APPLE__
+#ifndef __APPLE__
     // After 10.2 there is no need to clear the IOKit Master Port.
 #elif !defined(WIN32)
 	if (libirecovery_context != NULL) {
@@ -344,7 +347,7 @@
 			   unsigned char *data,
 			   uint16_t wLength, unsigned int timeout)
 {
-#ifdef __APPLE__
+#ifndef __APPLE__
     IOReturn err;
     IOUSBDevRequestTO req;
     
@@ -368,8 +371,30 @@
     return req.wLenDone;
 
 #elif !defined(WIN32)
-	return libusb_control_transfer(client->handle, bmRequestType, bRequest,
-				       wValue, wIndex, data, wLength, timeout);
+#ifndef __APPLE__
+	return libusb_control_transfer(client->handle, bmRequestType, bRequest, wValue, wIndex, data, wLength, timeout);
+#else
+	if (timeout <= 10) {
+		// pod2g: dirty hack for limera1n support.
+		IOReturn kresult;
+		IOUSBDevRequest req;
+		bzero(&req, sizeof(req));
+		//struct darwin_device_handle_priv *priv = (struct darwin_device_handle_priv *)client->handle->os_priv;
+		struct darwin_device_priv *dpriv = (struct darwin_device_priv *)client->handle->dev->os_priv;
+		req.bmRequestType     = bmRequestType;
+		req.bRequest          = bRequest;
+		req.wValue            = OSSwapLittleToHostInt16 (wValue);
+		req.wIndex            = OSSwapLittleToHostInt16 (wIndex);
+		req.wLength           = OSSwapLittleToHostInt16 (wLength);
+		req.pData             = data + LIBUSB_CONTROL_SETUP_SIZE;
+		kresult = (*(dpriv->device))->DeviceRequestAsync(dpriv->device, &req, (IOAsyncCallback1) dummy_callback, NULL);
+		usleep(5 * 1000);
+		kresult = (*(dpriv->device))->USBDeviceAbortPipeZero (dpriv->device);
+		return kresult == KERN_SUCCESS ? 0 : -1;
+	} else {
+		return libusb_control_transfer(client->handle, bmRequestType, bRequest, wValue, wIndex, data, wLength, timeout);
+	}
+#endif
 #else
 	DWORD count = 0;
 	BOOL bRet;
@@ -424,7 +449,7 @@
 			int length, int *transferred, unsigned int timeout)
 {
 	int ret;
-#ifdef __APPLE__
+#ifndef __APPLE__
     IOReturn err;
     
     err = (*client->handle.interface)->WritePipeTO(client->handle.interface, endpoint, data, length, timeout, timeout);
@@ -459,7 +484,7 @@
 int irecv_get_string_descriptor_ascii(irecv_client_t client, uint8_t desc_index,
 				      unsigned char *buffer, int size)
 {
-#ifdef __APPLE__
+#ifndef __APPLE__
 	irecv_error_t ret;
 	unsigned short langid = 0;
 	unsigned char data[255];
@@ -531,7 +556,7 @@
 
 irecv_error_t irecv_open(irecv_client_t * pclient)
 {
-#ifdef __APPLE__
+#ifndef __APPLE__
     io_iterator_t deviceIterator;
     io_service_t usbDevice;
     UInt8 numConfigurations;
@@ -764,7 +789,7 @@
 {
 	if (check_context(client) != IRECV_E_SUCCESS)
 		return IRECV_E_NO_DEVICE;
-#ifdef __APPLE__
+#ifndef __APPLE__
     IOReturn error;
     
     USB_LOG("Setting to device configuration %d.\n", configuration);
@@ -800,7 +825,7 @@
 {
 	if (check_context(client) != IRECV_E_SUCCESS)
 		return IRECV_E_NO_DEVICE;
-#ifdef __APPLE__
+#ifndef __APPLE__
     IOReturn err;
     UInt32 score;
     IOUSBFindInterfaceRequest req;
@@ -900,7 +925,7 @@
 
 irecv_error_t irecv_reset(irecv_client_t client)
 {
-#ifdef __APPLE__
+#ifndef __APPLE__
     IOReturn err;
     
     err = (*client->handle.device)->ResetDevice(client->handle.device);
@@ -1016,7 +1041,7 @@
 			event.type = IRECV_DISCONNECTED;
 			client->disconnected_callback(client, &event);
 		}
-#ifdef __APPLE__
+#ifndef __APPLE__
         if(client->handle.device) {
             if(client->handle.interface) {
                 (*client->handle.interface)->USBInterfaceClose(client->handle.interface);
