---
layout: post
title: Android通信之BLE
date: 2019-05-14
tags: [通信,android]
author: loren1994
category: blog
---

# Android 通信之BLE

##### 通信方式

* BLE蓝牙/传统蓝牙
* USB
* 局域网Socket
* 热点
* WebSocket
* Netty

等等...

##### BLE和BT

一般将蓝牙3.0之前的BR/EDR蓝牙称为传统蓝牙，而将蓝牙4.0规范下的LE蓝牙称为低功耗蓝牙。

蓝牙4.0标准包括传统蓝牙模块部分和低功耗蓝牙模块部分，是一个双模标准。低功耗蓝牙也是建立在传统蓝牙基础之上发展起来的，并区别于传统模块，最大的特点就是成本和功耗降低，应用于实时性要求比较高。

##### BLE蓝牙通信

现在手机上一般都是双模蓝牙，即包括低功耗蓝牙又包括传统蓝牙，传统蓝牙用到BluetoothSocket与socket类似，不在说明，此处重点为BLE蓝牙。

##### BLE基本概念

在BLE协议中，有两个角色，周边（Periphery）和中央（Central）。周边是数据的提供者，中央是数据的使用和处理者。在Android SDK里面，Android4.3以后手机可以作为中央使用；Android5.0以后手机才可以作为周边使用，即此时的手机可以作为BLE设备（如可穿戴设备、手环、智能锁、心率测量仪等）来为中央提供数据。 
一个中央可以同时连接多个周边，但一个周边某一时刻只能连接一个中央。 

Android BLE SDK的四个关键类如下： 

1、BluetoothGattServer作为周边来提供数据，BluetoothGattServerCallback返回周边的状态，更通俗的说，当中央有请求时，系统调用该抽象类的相应方法传递数据给周边。 

2、BluetoothGatt作为中央来使用和处理数据，BluetoohGattCallback返回中央的状态和周边提供数据，即周边反馈的数据通过该抽象类的相应方法传递到中央。 

##### 基本流程

现有两部手机A和B，一台接收，一台发送。

手机A(周边) => 开启蓝牙、打开GattServer并监听数据/状态等、添加自定义特征和描述的蓝牙服务、开始广播广告使得自定义的服务能被搜索到、通过GattServer收发数据。

手机B(中央) => 待A中的服务可被搜索后开始搜索、连接、打开监听、发送数据。

> 每一步操作之间最好设有一定间隔时间，否则操作可能会报错。

##### 使用方式

~~~~groovy
implementation 'com.clj.fastble:FastBleLib:2.3.2'
~~~~

使用FastBle库来操作，该库进行了一些函数封装，可一定程度上简化Api。

* 打开GattServer并监听

~~~~kotlin
mGattServer = bluetoothManager.openGattServer(this, callBack)
private val callBack = object : BluetoothGattServerCallback() {

        override fun onServiceAdded(status: Int, service: BluetoothGattService) {
            if (status == BluetoothGatt.GATT_SUCCESS) {
                log("添加服务成功！服务的uuid为" + service.uuid.toString())
            } else {
                log("添加服务失败！")
            }
            startAdvertise()
        }

        override fun onConnectionStateChange(device: android.bluetooth.BluetoothDevice, status: Int, newState: Int) {
            //2成功 0断开
            log("Ble连接状态为$newState")
            //保存BluetoothDevice对象用来往中央设备发送消息
            bleDev = if (newState == 2) device else null
        }

        override fun onCharacteristicReadRequest(device: android.bluetooth.BluetoothDevice,
                                                 requestId: Int, offset: Int, characteristic: BluetoothGattCharacteristic) {
            log("客户端来读取数据")
            mGattServer.sendResponse(device, requestId, GATT_SUCCESS, 0, characteristic.value)
        }

        override fun onCharacteristicWriteRequest(device: android.bluetooth.BluetoothDevice,
                                                  requestId: Int, characteristic: BluetoothGattCharacteristic, preparedWrite: Boolean,
                                                  responseNeeded: Boolean, offset: Int, value: ByteArray) {
            mGattServer.sendResponse(device, requestId, GATT_SUCCESS, 0, value)
            dataFromBle += String(value)
            log("接收到:${String(value)} \n 总:$dataFromBle")
        }

        override fun onDescriptorWriteRequest(device: BluetoothDevice?, requestId: Int, descriptor: BluetoothGattDescriptor?, preparedWrite: Boolean, responseNeeded: Boolean, offset: Int, value: ByteArray?) {
            super.onDescriptorWriteRequest(device, requestId, descriptor, preparedWrite, responseNeeded, offset, value)
            log("onDescriptorWriteRequest")
            mGattServer.sendResponse(device, requestId, 0, offset, value)
        }
}
~~~~

* 添加自定义服务

~~~~kotlin
private fun addService() {
        val bluetoothGattService = BluetoothGattService(UUID.fromString(WIFI_SERVICE_UUID), BluetoothGattService.SERVICE_TYPE_PRIMARY)
        val characteristic = BluetoothGattCharacteristic(UUID.fromString(WIFI_NOTIFY_UUID),
                BluetoothGattCharacteristic.PROPERTY_READ or
                        BluetoothGattCharacteristic.PROPERTY_WRITE or
                        BluetoothGattCharacteristic.PROPERTY_NOTIFY,
                BluetoothGattCharacteristic.PROPERTY_READ or
                        BluetoothGattCharacteristic.PROPERTY_WRITE or
                        BluetoothGattCharacteristic.PROPERTY_NOTIFY)
        val descriptor = BluetoothGattDescriptor(UUID.fromString(CONTROL_DESC_UUID), BluetoothGattDescriptor.PERMISSION_WRITE)
        descriptor.value = BluetoothGattDescriptor.ENABLE_NOTIFICATION_VALUE
        characteristic.addDescriptor(descriptor)
        bluetoothGattService.addCharacteristic(characteristic)
        mGattServer.addService(bluetoothGattService)
}
~~~~

* 开始广播广告

~~~~kotlin
private fun startAdvertise() {
        BleManager.getInstance().bluetoothAdapter.bluetoothLeAdvertiser
                .startAdvertising(createAdvSettings(true, 0), createFMPAdvertiseData(), adCallBack)
}

private fun createAdvSettings(connectable: Boolean, timeoutMillis: Int): AdvertiseSettings {
        val builder = AdvertiseSettings.Builder()
                .setAdvertiseMode(AdvertiseSettings.ADVERTISE_MODE_BALANCED)
                .setConnectable(connectable)
                .setTimeout(timeoutMillis)
                .setTxPowerLevel(AdvertiseSettings.ADVERTISE_TX_POWER_HIGH)
        return builder.build()
}

private fun createFMPAdvertiseData(): AdvertiseData {
        val builder = AdvertiseData.Builder()
                .setIncludeDeviceName(true)
        return builder.build()
}
~~~~

* 往中央设备发送消息

~~~~kotlin
private fun notifyGatt(device: BluetoothDevice, content: String) {
        var characteristic: BluetoothGattCharacteristic? = null
        mGattServer!!.services.forEach {
            it.characteristics.forEach { item ->
                if (item.uuid.toString().contentEquals(CONTROL_NOTIFY_UUID)) {
                    characteristic = item
                }
            }
        }
        characteristic?.let {
            it.value = "test".toByteArray()
            mGattServer!!.notifyCharacteristicChanged(device, it, false)
        }
}
~~~~

* 中央设备连接周边设备

~~~~kotlin
//bleDevice为扫描出来的ble设备对象
private fun connectBle() {
        BleManager.getInstance().connect(bleDevice, object : BleGattCallback() {
            override fun onStartConnect() {
                showLoadingDialog("连接中")
            }

            override fun onDisConnected(isActiveDisConnected: Boolean, device: BleDevice?, gatt: BluetoothGatt?, status: Int) {
                log("断开连接")
                //断开重连
                Handler().postDelayed({ connectBle() }, 300)
            }

            override fun onConnectSuccess(bleDevice: BleDevice, gatt: BluetoothGatt, status: Int) {
                toast("连接成功")
                log("连接成功")
                dismissLoadingDialog()
                gatt.requestConnectionPriority(BluetoothGatt.CONNECTION_PRIORITY_HIGH)
                Handler().postDelayed({ notifyBle() }, 300)
            }

            override fun onConnectFail(bleDevice: BleDevice?, exception: BleException) {
                dismissLoadingDialog()
                toast("连接失败")
                log("连接失败 - ${exception.description}")
                finish()
            }
        })
}
~~~~

* 打开监听

~~~~kotlin
private fun notifyBle() {
        BleManager.getInstance().notify(bleDevice, CONTROL_SERVICE_UUID, CONTROL_NOTIFY_UUID,
                object : BleNotifyCallback() {
                    override fun onNotifySuccess() {
                        log("打开通知成功")
                    }

                    override fun onNotifyFailure(exception: BleException) {
                        log("打开通知失败")
                    }

                    override fun onCharacteristicChanged(data: ByteArray) {
                        val result = String(data)
                        log("notify:$result")
                    }
                })
}
~~~~

* 向周边设备发送数据

~~~~kotlin
private fun sendData(content: String, isFinish: Boolean = false) {
        showLoadingDialog("发送中")
        BleManager.getInstance().write(bleDevice, CONTROL_SERVICE_UUID, CONTROL_NOTIFY_UUID, content.toByteArray(),
                object : BleWriteCallback() {
                    override fun onWriteSuccess(current: Int, total: Int, justWrite: ByteArray) {
                        dismissLoadingDialog()
                        log("发送数据到设备成功 - $current - $total - ${String(justWrite)}")
                        //分包发送
                        if (current == total) {
                            toast("发送成功")
                        }
                    }

                    override fun onWriteFailure(exception: BleException) {
                        dismissLoadingDialog()
                        toast("发送失败")
                        log("发送数据到设备失败: ${exception.description}")
                    }

                })
}
~~~~

> BLE单次只能发送20字节，超出则需要分包发送。

##### 注意点

* 周边要执行mGattServer!!.sendResponse后中央设备才会收到回应
* BLE只要是发数据单个包均有20字节限制
* 操作之间设置一定间隔时间
* 创建自定义服务时注意权限设置

