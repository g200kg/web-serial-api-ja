# Serial API 手引き書

このドキュメントは、 [Serial API](http://wicg.github.io/serial/) の説明です。これは、Webページがシリアルデバイスと通信できるようにするために提案された仕様です。  
<!--This document is an explainer for the [Serial API](http://wicg.github.io/serial/), a proposed specification for allowing a web page to communicate with a serial device.-->

## 目的

ユーザーは、特に教育、趣味、産業分野では、周辺機器をコンピューターに接続し、制御にカスタムソフトウェアを必要とします。 たとえば、ロボット工学は、学校でコンピュータプログラミングや電子工学を教えるためによく使用されます。 これには、コードをロボットにアップロードしたり、リモートで制御したりできるソフトウェアが必要です。 産業または愛好家の設定では、ミル、レーザーカッター、3Dプリンターなどの機器は、接続されたコンピューターで実行されているプログラムによって制御されます。 これらのデバイスは、多くの場合、シリアル接続を介して小さなマイクロコントローラーによって制御されます。
<!--Users, especially in the educational, hobbyist and industrial sectors, connect peripheral devices to their computers that require custom software to control. For example, robotics are often used to teach computer programming and electronics in schools. This requires software which can upload code to a robot and/or control it remotely. In an industrial or hobbyist setting a piece of equipment such as a mill, laser cutter or 3D printer is controlled by a program running on a connected computer. These devices are often controlled by small microcontrollers via a serial connection.-->

この制御ソフトが Web 技術を用いて構築されている例は数多くあります。例えば
<!--There are many examples of this control software being built using web technology. For example,-->

* [Arduino Create](https://create.arduino.cc/)
* [Betaflight Configurator](https://github.com/betaflight/betaflight-configurator)
* [Espruino IDE](http://espruino.com/ide)
* [MakeCode](https://www.microsoft.com/en-us/makecode)

場合によっては、これらの Web サイトは、ユーザーが手動でインストールしたネイティブエージェントアプリケーションを介してデバイスと通信します。 他の方法としては、アプリケーションは、Electronなどのフレームワークを介してパッケージ化されたネイティブアプリケーションで提供されます。 またその他のケースでは、ユーザーは、コンパイルされたアプリケーションを USB フラッシュドライブを介してデバイスにコピーするなど、更に追加の手順を実行する必要があります。
<!--In some cases these web sites communicate with the device through a native agent application that is manually installed by the user. In others the application is delivered in a packaged native application through a framework such as Electron. In others the user is required to perform an additional step such as copying a compiled application to the device via a USB flash drive.-->

これらすべての場合において、サイトとそれが制御しているデバイスとの間に直接通信を提供することにより、ユーザーエクスペリエンスが向上します。 ネイティブコンポーネントのインストールが必要なサイトの場合、これにより、タスクを実行するためにサイト作成者に付与される強力な機能の範囲が制限されるため、ユーザーのセキュリティとプライバシーも向上します。
<!--In all these cases the user experience would be improved by providing direct communication between the site and the device that it is controlling. For sites that require installation of a native component this will also improve user security and privacy by limiting the scope of powerful capabilities granted to the site author in order to perform the task.-->

### なぜWebBluetoothまたはWebUSBではないのですか？
<!--### Why not Web Bluetooth or WebUSB?-->

WebBluetooth および WebUSB API は、Bluetooth LowEnergyおよびUSB周辺機器への低レベルのアクセスを提供します。 シリアルインターフェースを備えたほとんどのデバイスが Bluetooth または USB であるなら、なぜ別の API が必要なのでしょうか？
<!--The Web Bluetooth and WebUSB APIs provide low level access to Bluetooth Low Energy and USB peripherals. If most devices with a serial interface are Bluetooth or USB why do we need a separate API?-->

 - すべてのシリアルデバイスが Bluetooth または USB デバイスであるとは限りません。 一部のプラットフォームには、 DE-9 コネクタ ( ほとんどの PC プラットフォーム ) またはシステムボードのヘッダー ( Raspberry Pi など ) としてシリアルインターフェイスを提供する UART が組み込まれています。  
<!--1. Not all serial devices are Bluetooth or USB devices. Some platforms still include a built-in UART providing a serial interface either as a DE-9 connector (most PC platforms) or as headers on the system board (for example, the Raspberry Pi).-->
  

 - ほとんどのオペレーティングシステムでは、アプリケーション（ユーザーエージェントを含む）は利用可能な最も高レベルの API を使用してデバイスと対話する事が必要です。たとえば、 USB デバイスが標準の USB CDC-ACM インターフェイスクラスを実装している場合、組み込みのクラスドライバーはそのインターフェイスを使用して、仮想シリアルポートインターフェイスを提供します。 Web USB API の実装からは USB インターフェースが要求されるため、代替としてそれを要求できません。 デバイスには、システムのシリアルポート API を介してアクセスする必要があります。
<!--2. Most operating systems require applications (including user agents) to interact with devices using the highest-level API available. For example, if a USB device implements the standard USB CDC-ACM interface class then a built-in class driver will claim that interface and provide a virtual serial port interface. Because the USB interface has been claimed an implementation of the WebUSB API cannot claim it instead. The device must be accessed through the system's serial port API.-->

## 潜在性に関する API

サイトがシリアルデバイスに接続する前に、アクセスを要求する必要があります。サイトがすべての潜在的なデバイスのサブセットとの通信をサポートしている場合は、USB ベンダー ID のような特定のプロパティに一致するデバイスに選択可能なデバイスのセットを制限するフィルタを提供することができます。
<!--Before a site can connect to a serial device it must request access. If a site only supports communciating with a subset of all potential devices then it can provide a filter which will limit the set of selectable devices to those matching certain properties such as a USB vendor ID.-->

```javascript
const filter = {
  usbVendorId: 0x2341 // Arduino SA
};

try {
  const port = await navigator.serial.requestPort({filters: [filter]});
  // Continue connecting to |port|.
} catch (e) {
  // Permission to access a device was denied implicitly or explicitly by the user.
}
```

`SerialPort` インスタンスにアクセスする事で、サイトはポートへの接続を開くことができます。  `open()` のほとんどのパラメータはオプションですが、適切なデフォルトがないためボーレートは必要になります。 開発者のあなたは、デバイスが通信する際に期待する速度を知っている必要があります。
<!--With access to a `SerialPort` instance the site may now open a connection to the port. Most parameters to `open()` are optional however the baud rate is required as there is no sensible default. You as the developer must know the rate at which your device expects to communicate.-->

```javascript
await port.open({ baudRate: /* pick your baud rate */ });
```

この時点で、`readable` 属性と `writable` 属性には、接続されたデバイスとの間でデータを送受信するために使用できる [`ReadableStream`](https://streams.spec.whatwg.org/#rs-class) と [`WritableStream`](https://streams.spec.whatwg.org/#ws-class) が設定されます。
<!--At this point the `readable` and `writable` attributes are populated with a [`ReadableStream`](https://streams.spec.whatwg.org/#rs-class) and [`WritableStream`](https://streams.spec.whatwg.org/#ws-class) that can be used to receive data from and send data to the connected device.-->

この例では、 [Hayes コマンドセット](https://en.wikipedia.org/wiki/Hayes_command_set) に似たプロトコルを実装するデバイスを想定しています。 コマンドは ASCII でエンコードされているため、 [`TextEncoder`](https://encoding.spec.whatwg.org/#interface-textencoder) と [`TextDecoder`](https://encoding.spec.whatwg.org/#interface-textdecoder) を使用して `SerialPort` のストリームで使用される `Uint8Array` を文字列との間で変換します。 
<!--In this example we assume a device implementing a protocol inspired by the [Hayes command set](https://en.wikipedia.org/wiki/Hayes_command_set). Since commands are encoded in ASCII a [`TextEncoder`](https://encoding.spec.whatwg.org/#interface-textencoder) and [`TextDecoder`](https://encoding.spec.whatwg.org/#interface-textdecoder) are used to translate the `Uint8Array`s used by the `SerialPort`'s streams to and from strings.-->

```javascript
const encoder = new TextEncoder();
const writer = port.writable.getWriter();
writer.write(encoder.encode("AT"));

const decoder = new TextDecoder();
const reader = port.readable.getReader();
const { value, done } = await reader.read();
console.log(decoder.decode(value));
// Expected output: OK
```

ポートを閉じる前に、読み取り可能および書き込み可能なストリームのロックを解除する必要があります。
<!--The readable and writable streams must be unlocked before the port can be closed.-->

```javascript
writer.releaseLock();
reader.releaseLock();
await port.close();
```

ストリームから単一のチャンクを読み取るのではなく、次のようにループを使用して継続的に読み取るコードがよく使われます。
<!--Rather than reading a single chunk from the stream code will often read continuously using a loop like this,-->

```javascript
const reader = port.readable.getReader();
while (true) {
  const { value, done } = await reader.read();
  if (done) {
    // |reader| has been canceled.
    break;
  }
  // Do something with |value|...
}
reader.releaseLock();
```

この場合、 `port.readable` はストリームがエラーに遭遇するまでロックが解除されませんので、ポートを閉じるにはどうすればいいのでしょうか？ `reader` で `cancel()` を呼び出すと、 `read()` で返された `Promise` が `{ value: undefined, done: true }` で即座に解決されます。これにより、上記のコードがループから抜け出してストリームのロックを解除し、ポートを閉じることができるようになります。
<!--In this case `port.readable` will not be unlocked until the stream encounters an error, so how do you close the port? Calling `cancel()` on `reader` will cause the `Promise` returned by `read()` to resolve immediately with `{ value: undefined, done: true }`. This will cause the code above to break out of the loop and unlock the stream so that the port can be closed,-->

```javascript
await reader.cancel();
await port.close();
```

シリアルポートは、バッファオーバーフロー、フレーミングエラー、パリティエラーなどの条件で、致命的ではない読み込みエラーを生成することがあります。これらのエラーは `read()` メソッドの例外としてスローされ、 `ReadableStream`  がエラーになる原因となります。エラーが致命的でない場合は、 `port.readable`  はエラーの直後に新しい `ReadableStream`  に置き換えられます。上の例を拡張してこれらのエラーを処理するために、別のループを追加しました。
<!--A serial port may generate one of a number of non-fatal read errors for conditions such as buffer overflow, framing or parity errors. These are thrown as exceptions from the `read()` method and cause the `ReadableStream` to become errored. If the error is non-fatal then `port.readable` is immediately replaced by a new `ReadableStream` that picks up right after the error. To expand the example above to handle these errors another loop is added,-->

```javascript
while (port.readable) {
  const reader = port.readable.getReader();
  while (true) {
    let value, done;
    try {
      ({ value, done } = await reader.read());
    } catch (error) {
      // Handle |error|...
      break;
    }
    if (done) {
      // |reader| has been canceled.
      break;
    }
    // Do something with |value|...
  }
  reader.releaseLock();
}
```

USBデバイスが取り外されるなどの致命的なエラーが発生した場合、 `port.readable` は `null` に設定されます。
<!--If a fatal error occurs, such as a USB device being removed, then `port.readable` will be set to `null`.-->

先ほどの例に戻りますが、常に ASCII テキストを生成するデバイスの場合、 [`transformStream`](https://streams.spec.whatwg.org/#ts) を使用することで `encode()` と `decode()` の明示的な呼び出しを削除することができます。この例では、 `writer` は [`TextEncoderStream`](https://encoding.spec.whatwg.org/#interface-textencoderstream) から、`reader` は [`TextDecoderStream`](https://encoding.spec.whatwg.org/#interface-textdecoderstream) から作成します。 `pipeTo()` メソッドは、これらの変換をポートに接続するために使用されます。
<!--Revisiting the earlier example, for a device that always produces ASCII text the explicit calls to `encode()` and `decode()` can be removed through the use of [`TransformStream`](https://streams.spec.whatwg.org/#ts)s. In this example `writer` comes from a [`TextEncoderStream`](https://encoding.spec.whatwg.org/#interface-textencoderstream) and `reader` comes from a [`TextDecoderStream`](https://encoding.spec.whatwg.org/#interface-textdecoderstream). The `pipeTo()` method is used to connect these transforms to the port.-->

```javascript
const encoder = new TextEncoderStream();
const writableStreamClosed = encoder.readable.pipeTo(port.writable);
const writer = encoder.writable.getWriter();
writer.write("AT");

const decoder = new TextDecoderStream();
const readableStreamClosed = port.readable.pipeTo(decoder.writable);
const reader = decoder.readable.getReader();
const { value, done } = await reader.read();
console.log(value);
// Expected output: OK
```

トランスフォームストリームを経由してパイプを行う場合、ポートを閉じることはより複雑になります。 `reader` や `writer` を閉じると、エラーがトランスフォームストリームを通って下層にあるポートに伝搬してしまいます。しかし、この伝搬はすぐには起こりません。 `port.readable` と `port.writable` がアンロックされたことを検出するために、新しい `writableStreamClosed` と `readableStreamClosed` のプロミスが必要になります。 `reader` をキャンセルするとストリームが中断されるので、結果として発生するエラーをキャッチして無視しなければなりません。
<!--When piping through a transform stream closing the port becomes more complicated. Closing `reader` or `writer` will cause an error to propagate through the transform streams to the underlying port. However, this propagation doesn't happen immediately. The new `writableStreamClosed` and `readableStreamClosed` promises are required to detect when `port.readable` and `port.writable` have been unlocked. Since canceling `reader` causes the stream to be aborted the resulting error must be caught and ignored,-->

```javascript
writer.close();
await writableStreamClosed;
reader.cancel();
await readableStreamClosed.catch(reason => {});
await port.close();
```

シリアルポートには、デバイス検出とフロー制御のための追加信号が多数含まれており、これらは明示的に問い合わせて設定することができます。例えば、 Arduino のようないくつかのデバイスは、 [Data Terminal Ready](https://en.wikipedia.org/wiki/Data_Terminal_Ready)  (DTR) 信号がトグルされるとプログラミングモードになります。
<!--Serial ports include a number of additional signals for device detection and flow control which can be queried and set explicitly. As an example, some devices like the Arduino will enter a programming mode if the [Data Terminal Ready](https://en.wikipedia.org/wiki/Data_Terminal_Ready) (DTR) signal is toggled.-->

```javascript
await port.setSignal({ dataTerminalReady: false });
await new Promise(resolve => setTimeout(200, resolve));
await port.setSignal({ dataTerminalReady: true });
```

シリアルポートが USB デバイスによって供給されている場合、そのデバイスはシステムに接続されたり切断されたりします。サイトがポートへのアクセス許可を得ると、これらのイベントを受信し、現在アクセスしている接続デバイスのセットを照会することができます。
<!--If a serial port is provided by a USB device then that device may be connected or disconnected from the system. Once a site has permission to access a port it can receive these events and query for the set of connected devices it currently has access to.-->

```javascript
// Check to see what ports are available when the page loads.
document.addEventListener('DOMContentLoaded', async () => {
  let ports = await navigator.serial.getPorts();
  // Populate the UI with options for the user to select or automatically
  // connect to devices.
});

navigator.serial.addEventListener('connect', e => {
  // Add |e.target| to the UI or automatically connect.
});

navigator.serial.addEventListener('disconnect', e => {
  // Remove |e.target| from the UI. If the device was open the disconnection can
  // also be observed as a stream error.
});
```

**注：**Chrome 89より前のバージョンでは、 `connect` および `disconnect` イベントでは、 `navigator.serial` でカスタム `SerialConnectionEvent` オブジェクトが発行され、影響を受ける `SerialPort` インターフェイスが `port` 属性として使用可能でした。Chrome 89 以降では、一般的な `Event` オブジェクトが `SerialPort` インターフェイス自体で発行されます。 イベントリスナーは、これらのイベントが `SerialPort` インターフェイスから `Serial` インターフェイスにバブリングされるため、 `navigator.serial` にイベントリスナーを登録したままでも、イベントリスナーを登録することができます。以前のバージョンとの互換性を保つために、 "`e.port || e.target`" という式を使用して `port` 属性（存在する場合）または `target` 属性（存在しない場合）のいずれかを取得できます。

<!--**Note:** Prior to Chrome 89 the `connect` and `disconnect` events fired a custom `SerialConnectionEvent` object at `navigator.serial` with the affected `SerialPort` interface available as the `port` attribute. 
In Chrome 89 and above a generic `Event` object is fired at the `SerialPort` interface itself. 
An event listener can still register an event listener on `navigator.serial` as these events bubble from the `SerialPort` interface to the `Serial` interface. For compatibility with the earlier version the expression "`e.port || e.target`" can be used to get either the `port` attribute (if present) or the `target` attribute (if not).-->

## Security considerations

This API poses similar a security risk to the Web Bluetooth and WebUSB APIs and so lessons from those are applicable here. The primary threats are:

* Exploitation of a device’s capabilities by malicious code that has been granted access.
* Installation of malicious firmware on a device that can be used to attack the host to which it is connected.
* Malicious code injected into a site which has been granted access doing any of the above.

The primary mitigation is a permission model that grants access to only a single device at a time. In response to the prompt displayed by a call to `requestDevice()` the user must take active steps to select a particular device. This prevents drive-by attacks against connected devices. Implementations may also give the users a visual indication that a page is currently communicating with a device and controls for revoking that permission at any time.

The user agent must also require the page to be served from a secure origin in order to prevent malicious code from being injected by a network-based attacker. Secure delivery of code does not indicate that the code is trustworthy but is a minimum requirement for ensuring that other security decisions being made based on the site's origin are effective. The user agent must also prevent cross-origin iframes from using the API unless explicitly granted permission by the embedding page through [Feature Policy](https://w3c.github.io/webappsec-feature-policy/). This mitigates most malicious code injection attacks unless the trusted site itself is compromised.

The remaining concern is the exploitation of a connected device through a phishing attack that convinces the user to grant a malicious site access to a device. These attacks exploit the trust that a device typically places in the host computer it is connected to and can be used to either exploit the device’s capabilities as designed or to install malicious firmware on the device that will in turn attack the host computer. There is no mechanism that will completely prevent this type of attack because the meaning of the data sent from a page to the device is opaque to the user agent. Any attempt to block a particular type of data from being sent will be met by workarounds on the part of device manufacturers who nevertheless want to send this type of data to their devices.

User agents may implement additional settings to mitigate potential phishing attacks:

* Settings to allow the user to change the default permission setting for this API from “ask” (meaning that the permission prompt is shown) to “block” which prevents the prompt from being displayed entirely.
* Enterprise policy settings so that concerned systems administrators can apply this default throughout their organization. This default could be overridden for particular trusted origins.
* A list of the USB and Bluetooth device IDs for hardware which is known to be exploitable can be distributed to user agents by the vendor and a centralized registry similar the ones maintained for [Web Bluetooth](https://github.com/WebBluetoothCG/registries) and [WebUSB](https://github.com/WICG/webusb/blob/master/blocklist.txt). Connections to these devices would be blocked.

This final mitigation is more difficult to apply to this API. There are a number of reasons for this. First, it is difficult to define what “exploitable” means. For example, this API will allow a page to upload firmware to Arduino boards. This is in fact a major use case for this API as these devices are common in the educational and hobbyist markets. These boards do not implement firmware signature verification and so can easily be turned into a malicious device. Should they be blocked? No, Arduino users have to accept this risk.

Also, unlike USB and Bluetooth devices, it is difficult to obtain the true identity of a serial device as it may be connected to directly to the host via a a DB-25, DE-9 or RJ-45 connector for which there is no handshake to establish identity, or through a generic USB- or Bluetooth-to-serial adapters.

## Privacy considerations

Serial devices contain two kinds of sensitive information,

1. When the device is a USB or Bluetooth device there are identifiers such as the vendor and product IDs (which identify the make and model) as well as a serial number or MAC address.
2. Additional identifiers may be available through commands sent via the serial port. The device may also store other private information which may or may not be considered private.

For the same reasons mentioned in the “Security considerations” section reguarding preventing a device from being programmed with malicious firmware it is impractical to prevent a page from accessing this information once it has been granted access. Instead the permission model gives the user control over exactly which devices a page has access to in the first place. A page cannot proactively enumerate the devices that could be chosen. This is similar to the file picker UI. A site cannot arbitrarily access the filesystem, only the files that have been chosen by the user. Once a file has been selected the site has access to the complete file. The user agent can also notify the user in real time when a page is using these permissions with some kind of indicator.
