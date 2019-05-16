---
layout: post
title: Android通信之USB
date: 2019-05-15
tags: [usb通信,android]
author: loren1994
category: blog
---

# Android通信之USB

##### 通信方式

* BLE蓝牙/传统蓝牙
* USB
* 局域网Socket
* 热点
* WebSocket
* Netty

等等...

##### USB通信介绍

即插即用，可热插拔。

USB2.0 ：理论速度是每秒480Mbps（约每秒60MB） 
USB3.0 ：理论速度能够达到每秒5Gbps（约为每秒625MB）

USB2.0总线提供最大达5v电压、500mA电流，USB3.0 可达1A。大部分USB外设无需单独的供电系统。 

通信包括USB主机(USB HOST)、 USB设备(USB DEVICE)。

android支持USB accessory模式和USB host模式。通过这两种模式，android支持各种各样的USB 外围设备和USB 配件（硬件需要实现android配件协议）。

##### OTG协议

OTG是On-The-Go的缩写。随着USB技术的发展，使得PC和周边设备能够通过简单的方式、适度的制造成本，将各种数据传输速度的设备连接在一起。但都是通过USB连接到PC，并在PC的控制下进行数据交换。这种方便的交换方式，一旦离开了PC，各设备间无法利用USB口进行操作，因为没有一个从设备能够充当PC一样的主机。OTG技术就是在没有Host的情况下，实现设备间的数据传送。

##### USB HOST

Android工作在USB Host模式下，则连接到Android上的USB设备把Android类似的看作是一台主机，例如将鼠标、键盘插入则可以使用键盘、鼠标来操作Android系统。

做以下设置可将手机作为HOST端

~~~~xml
//manifest
<uses-permission android:name="android.hardware.usb.host" />
~~~~

##### USB DEVICE

将android手机设置为USB accessory模式

~~~~xml
//manifest
<uses-feature android:name="android.hardware.usb.accessory" />
 <!--过滤USB设备  product-id vendor-id-->
<activity android:name=".UsbAccessoryActivity">
            <intent-filter>
                <action android:name="android.hardware.usb.action.USB_ACCESSORY_ATTACHED" />
            </intent-filter>
            <meta-data
                android:name="android.hardware.usb.action.USB_ACCESSORY_ATTACHED"
                android:resource="@xml/accessory_filter" />
</activity>
~~~~

xml/accessory_filter.xml

~~~~xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <usb-accessory
        manufacturer="Google, Inc."
        model="loren"
        type="demo"
        version="1.0"/>
</resources>
~~~~

##### 基本流程

HOST => 订阅插拔广播、UsbManager枚举连接的设备、根据设备vendorId和productId找出指定设备、请求usb权限、打开Accessory模式、寻找设备接口、分配IN|OUT端点、打开连接通道、循环接收消息

ACCESSORY => 订阅插拔广播、申请权限、打开设备、循环接收

##### 使用

订阅插拔广播

~~~~kotlin
 val filter = IntentFilter(ACTION_USB_PERMISSION)
 registerReceiver(usbReceiver, filter)
 val filter1 = IntentFilter(UsbManager.ACTION_USB_ACCESSORY_DETACHED)
 registerReceiver(usbReceiver, filter1)
~~~~

枚举连接设备

~~~~~kotlin
private fun enumerateDevice(mUsbManager: UsbManager?) {
        log("开始进行枚举设备!")
        if (mUsbManager == null) {
            log("创建UsbManager失败，请重新启动应用！")
            return
        } else {
            val deviceList = mUsbManager.deviceList
            if (!deviceList.isEmpty()) {
                val deviceIterator = deviceList.values.iterator()
                while (deviceIterator.hasNext()) {
                    val device = deviceIterator.next()
                    log("deviceInfo: ${device.vendorId} , ${device.productId}")
                    usbDevice = device
                    if (mUsbManager.hasPermission(usbDevice)) {
                        initAccessory(usbDevice)
                    } else {
                        val mPermissionIntent = PendingIntent.getBroadcast(this, 0, Intent(ACTION_USB_PERMISSION), 0)
                        mUsbManager.requestPermission(usbDevice, mPermissionIntent)
                    }
                }
            } else {
                log("device list 为空")
            }
        }
}
~~~~~

请求权限并打开

~~~~kotlin
val mPermissionIntent = PendingIntent.getBroadcast(this, 0, 
		Intent(ACTION_USB_PERMISSION), 0)
mUsbManager.requestPermission(usbDevice, mPermissionIntent)
//打开
private fun initAccessory(usbDevice: UsbDevice) {
        val usbDeviceConnection = mUsbManager.openDevice(usbDevice)
        if (usbDeviceConnection == null) {
            log("请连接USB")
            return
        }
        //根据AOA协议打开Accessory模式
        initStringControlTransfer(usbDeviceConnection, 0, "Google, Inc.") // MANUFACTURER
        initStringControlTransfer(usbDeviceConnection, 1, "loren") // MODEL
        initStringControlTransfer(usbDeviceConnection, 2, "loren desc") // DESCRIPTION
        initStringControlTransfer(usbDeviceConnection, 3, "1.0") // VERSION
        initStringControlTransfer(usbDeviceConnection, 4, "http://www.android.com") // URI
        initStringControlTransfer(usbDeviceConnection, 5, "0123456789") // SERIAL
        usbDeviceConnection.controlTransfer(0x40, 53, 0, 0, byteArrayOf(), 0, 100)
        usbDeviceConnection.close()
        getDeviceInterface()
}
~~~~

寻找设备接口

~~~~kotlin
//寻找设备接口
private fun getDeviceInterface() {
        log("interfaceCounts : ${usbDevice.interfaceCount}")
        for (i in 0 until usbDevice.interfaceCount) {
            val intf = usbDevice.getInterface(i)
            if (i == 0) {
                interface1 = intf
                assignEndpoint(intf)
                openDevice(intf)
            }
        }
}
~~~~

分配端点IN|OUT

~~~~kotlin
private fun assignEndpoint(mInterface: UsbInterface) {
        for (i in 0 until mInterface.endpointCount) {
            val ep = mInterface.getEndpoint(i)
            // look for bulk endpoint
            if (ep.type == UsbConstants.USB_ENDPOINT_XFER_BULK) {
                if (ep.direction == UsbConstants.USB_DIR_OUT) {
                    epBulkOut = ep
                    println("""Find the BulkEndpointOut,index:$i,使用端点号：${epBulkOut!!.endpointNumber}""")
                } else {
                    epBulkIn = ep
                    println("""Find the BulkEndpointIn:index:$i,使用端点号：${epBulkIn!!.endpointNumber}""")
                }
            }
        }
}
~~~~

建立连接

~~~~kotlin
private fun openDevice(mInterface: UsbInterface?) {
        var conn: UsbDeviceConnection? = null
        if (mUsbManager.hasPermission(usbDevice)) {
            conn = mUsbManager.openDevice(usbDevice)
        } else {
            log("无权限")
            val mPermissionIntent = PendingIntent.getBroadcast(this, 0, Intent(""), 0)
            mUsbManager.requestPermission(usbDevice, mPermissionIntent)
        }
        if (conn == null) {
            return
        }

        if (conn.claimInterface(mInterface, true)) {
            usbDeviceConnection = conn
            // 到此你的android设备已经连上设备
            if (usbDeviceConnection != null)
                log("open设备成功！")
            val mySerial = usbDeviceConnection!!.serial
            log("设备serial number：$mySerial")
        } else {
            log("无法打开连接通道")
            conn.close()
        }
        loopReceiverMessage()
}
~~~~

循环接收

~~~~kotlin
private fun loopReceiverMessage() {
        mThreadPool.execute {
            while (isReceiverMessage) {
                if (usbDeviceConnection != null && epBulkIn != null) {
                    val i = usbDeviceConnection!!.bulkTransfer(epBulkIn, mBytes, mBytes.size, 3000)
                    if (i > 0) {
                        //mBytes
                    }
                }
            }
        }
}
~~~~

HOST向ACCESSORY发送消息

~~~~kotlin
private fun sendMessageToPoint(buffer: ByteArray) {
        if (null == usbDeviceConnection) return
        mThreadPool.execute {
            val i = usbDeviceConnection!!.bulkTransfer(epBulkOut, buffer, buffer.size, 3000)
            if (i > 0) {
                log("发送成功")
            } else {
                log("发送失败")
            }
        }
}
~~~~

ACCESSORY打开设备

~~~~kotlin
private fun openAccessory(usbAccessory: UsbAccessory) {
        mParcelFileDescriptor = mUsbManager.openAccessory(usbAccessory)
        if (mParcelFileDescriptor != null) {
            log("打开成功")
            val fileDescriptor = mParcelFileDescriptor!!.fileDescriptor
            mFileInputStream = FileInputStream(fileDescriptor)
            mFileOutputStream = FileOutputStream(fileDescriptor)
            mThreadPool.execute {
                var i = 0
                while (i >= 0) {
                    try {
                        i = mFileInputStream!!.read(mBytes)
                    } catch (e: Exception) {
                        e.printStackTrace()
                        break
                    }
                    if (i > 0) {
                        log("接收到消息:${String(mBytes, 0, i)}")
                    }
                }
            }

        } else {
            log("mParcelFileDescriptor == null")
        }
}
~~~~

ACCESSORY向HOST发送消息

~~~~
mThreadPool.execute {
                try {
                    mFileOutputStream?.write("服务端发送消息".toByteArray())
                } catch (e: IOException) {
                    e.printStackTrace()
                }
}
~~~~

##### 注意

* 设备选择和端口的选择

项目中遇到过用第二个端口发送导致设备逻辑插拔，用第一个正常发送/接收

* 单次发送长度不能超过16384字节