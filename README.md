# BLESSED for Android - BLE made easy

[![](https://jitpack.io/v/weliem/blessed-android.svg)](https://jitpack.io/#weliem/blessed-android)

BLESSED is a very compact Bluetooth Low Energy (BLE) library for Android 5 and higher, that makes working with BLE on Android very easy. It takes care of many aspects of working with BLE you would normally have to take care of yourself like:

* *Queueing commands*, so you can don't have to wait anymore for the completion of a command before issueing the next command
* *Bonding*, so you don't have to do anything in order to robustly bond devices
* *Easy scanning*, so you don't have to setup complex scan filters
* *Managing threading*, so you don't have to worry about on which thread to issue commands or receive results 
* *Threadsafe*, so you don't see weird threading related issues anymore
* *Workarounds for some known Android bugs*, so you don't have to research any workarounds
* *Higher abstraction methods for convenience*, so that you don't have to do a lot of low-level management to get stuff done

The library consists of 3 core classes and 2 callback abstract classes:
1. `BluetoothCentral`, and it companion abstract class `BluetoothCentralCallback`
2. `BluetoothPeripheral`, and it's companion abstract class `BluetoothPeripheralCallback`
3. `BluetoothBytesParser`

The `BluetoothCentral` class is used to scan for devices and manage connections. The `BluetoothPeripheral` class is a replacement for the standard Android `BluetoothDevice` and `BluetoothGatt` classes. It wraps all GATT related peripheral functionality. The `BluetoothBytesParser` class is a utility class that makes parsing byte arrays easy.

The BLESSED library was inspired by CoreBluetooth on iOS and provides the same level of abstraction, but at the same time it also stays true to Android by keeping most methods the same and allowing you to work with the standard classes for Services, Characteristics and Descriptors. If you already have developed using CoreBluetooth you can very easily port your code to Android using this library.

## Scanning

There are 4 different scanning methods:

```java
public void scanForPeripherals()
public void scanForPeripheralsWithServices(UUID[] serviceUUIDs)
public void scanForPeripheralsWithNames(String[] peripheralNames)
public void scanForPeripheralsWithAddresses(String[] peripheralAddresses)
```

They all work in the same way and take an array of either service UUIDs, peripheral names or mac addresses. So in order to setup a scan for a device with the Bloodpressure service and connect to it, you do:

```java
private final BluetoothCentralCallback bluetoothCentralCallback = new BluetoothCentralCallback() {
        @Override
        public void onDiscoveredPeripheral(BluetoothPeripheral peripheral, ScanResult scanResult) {
            central.stopScan();
            central.connectPeripheral(peripheral, peripheralCallback);
        }
};

// Create BluetoothCentral and receive callbacks on the main thread
BluetoothCentral central = BluetoothCentral(getApplicationContext(), bluetoothCentralCallback, new Handler(Looper.getMainLooper()));

// Define blood pressure service UUID
UUID BLOODPRESSURE_SERVICE_UUID = UUID.fromString("00001810-0000-1000-8000-00805f9b34fb");

// Scan for peripherals with a certain service UUID
central.scanForPeripheralsWithServices(new UUID[]{BLOODPRESSURE_SERVICE_UUID});
```
**Note** Only 1 of these 4 types of scans can be active at one time! So call `stopScan()` before calling another scan.

## Connecting to devices

There are 2 ways to connect to a device:
```java
public void connectPeripheral(BluetoothPeripheral peripheral, BluetoothPeripheralCallback peripheralCallback)
public void autoConnectPeripheral(BluetoothPeripheral peripheral, BluetoothPeripheralCallback peripheralCallback)
```

The method `connectPeripheral` will try to immediately connect to a device that has already been found using a scan. This method will time out after 30 seconds or less depending on the device manufacturer. 

The method `autoConnectPeripheral` is for re-connecting to known devices for which you already know the device's mac address. The BLE stack will automatically connect to the device when it sees it in its internal scan. Therefore, it may take longer to connect to a device but this call will never time out! So you can issue the autoConnect command and the device will be connected whenever it is found. This call will **also work** when the device is not cached by the Android stack, as BLESSED takes care of it!

If you know the mac address of your peripheral you can obtain a `BluetoothPeripheral` object using:
```java
BluetoothPeripheral peripheral = central.getPeripheral("CF:A9:BA:D9:62:9E");
```

After issuing a connect call, you will receive one of the following callbacks:
```java
public void onConnectedPeripheral(BluetoothPeripheral peripheral)
public void onConnectionFailed(BluetoothPeripheral peripheral, int status)
public void onDisconnectedPeripheral(BluetoothPeripheral peripheral, int status)
```

## Service discovery

The BLESSED library will automatically do the service discovery for you and once it is completed you will receive the following callback:

```java
public void onServicesDiscovered(BluetoothPeripheral peripheral)
```
In order to get the services you can use methods like `getServices()` or `getService(UUID)`. In order to get hold of characteristics you can call `getCharacteristic(UUID)` on the BluetoothGattService object or call `getCharacteristic()` on the BluetoothPeripheral object.

This callback is the proper place to start enabling notifications or read/write characteristics.

## Reading and writing

Reading and writing to characteristics is done using the following methods:

```java
public boolean readCharacteristic(BluetoothGattCharacteristic characteristic)
public boolean writeCharacteristic(BluetoothGattCharacteristic characteristic, byte[] value, int writeType)
```

Both methods are asynchronous and will be queued up. So you can just issue as many read/write operations as you like without waiting for each of them to complete. You will receive a callback once the result of the operation is available.
For read operations you will get a callback on:

```java
public void onCharacteristicUpdate(BluetoothPeripheral peripheral, byte[] value, BluetoothGattCharacteristic characteristic)
```
And for write operations you will get a callback on:
```java
public void onCharacteristicWrite(BluetoothPeripheral peripheral, byte[] value, BluetoothGattCharacteristic characteristic, final int status)

```

In these callbacks, the *value* parameter is the threadsafe byte array that was received. Use this value instead of the value that is part of the BluetoothGattCharacteristic object since that one may have changed in the mean time because of incoming notifications or write operations.

## Turning notifications on/off

BLESSED provides a convenience method `setNotify` to turn notifications on or off. It will perform all the necessary operations like writing to the Client Characteristic Configuration descriptor for you. So all you need to do is:

```java
// See if this peripheral has the Current Time service
if(peripheral.getService(CTS_SERVICE_UUID) != null) {
     BluetoothGattCharacteristic currentTimeCharacteristic = peripheral.getCharacteristic(CTS_SERVICE_UUID, CURRENT_TIME_CHARACTERISTIC_UUID);
     peripheral.setNotify(currentTimeCharacteristic, true);
}
```

Since this is an asynchronous operation you will receive a callback that indicates success or failure:

```java
@Override
public void onNotificationStateUpdate(BluetoothPeripheral peripheral, BluetoothGattCharacteristic characteristic, int status) {
     if( status == GATT_SUCCESS) {
          if(peripheral.isNotifying(characteristic)) {
               Log.i(TAG, String.format("SUCCESS: Notify set to 'on' for %s", characteristic.getUuid()));
          } else {
               Log.i(TAG, String.format("SUCCESS: Notify set to 'off' for %s", characteristic.getUuid()));
          }
     } else {
          Log.e(TAG, String.format("ERROR: Changing notification state failed for %s", characteristic.getUuid()));
     }
}
```
When notifications arrive you will receive a callback on:

```java
public void onCharacteristicUpdate(BluetoothPeripheral peripheral, byte[] value, BluetoothGattCharacteristic characteristic)
```

## Example application

An example application is provided in the repo. It shows how to connect to Blood Pressure meters, read the data and show it on screen.

## Acknowledgements

BLESSED is the result of learning from many others about how to do BLE. Here are some references to material that helped me develop BLESSED:

* Stuart Kent's [presentation on BLE](https://www.stkent.com/2017/09/18/ble-on-android.html)
* [Emil's posts](https://stackoverflow.com/users/556495/emil) on StackOverflow
* The [Android BLE Library](https://github.com/NordicSemiconductor/Android-BLE-Library) by Nordic Semiconductor
* The [RxAndroidBle library](https://github.com/Polidea/RxAndroidBle) by Polidea

