### The problem:

> Would it be possible to implement (for WSS WebSockets) the same logic used for HTTPS when _AutoSelectCertificateForUrls_ policy is enabled? Which wouldn’t result in an UI pop-up.

**Context:** [crbug.com/993907](https://bugs.chromium.org/p/chromium/issues/detail?id=993907#c26)

### Analysis

- Main classes involved:
![Class diagram with the main classes involved](https://github.com/nickdiego/chromium-docs/raw/main/wss-client-certs/chromium-wss-vs-https-client-cert-classes.png)

- For regular http/https request, [network::URLLoader](https://source.chromium.org/chromium/chromium/src/+/main:services/network/url_loader.h;l=95;drc=e175ac54eff056647c204feef699536863d04afc) is used. Which encapsulates a bunch of things, including the IPC interaction with NetworkService (that tipically lives in a separate process), which ultimately uses a [net::URLRequest](https://source.chromium.org/chromium/chromium/src/+/main:net/url_request/url_request.h;l=85;drc=256182eb99a7e9de19375dc250df8b3d79e0bda8) to drive the connection, as well as SSL auth/certificate handling delegated to browser process (see [network::mojom::URLLoaderNetworkServiceObserver](https://source.chromium.org/chromium/chromium/src/+/main:services/network/public/mojom/url_loader_network_service_observer.mojom;l=83;drc=169c6cc102b39295a5bfe2f2a176b42b1c2fe2c4) in the class diagram) through mojo endpoints, etc.
- OTOH, for WebSockets, [network::WebSocket](https://source.chromium.org/chromium/chromium/src/+/main:services/network/websocket.h;l=54;drc=861e5739e4a59f36e209efef954193030ca234a9) is the "equivalent" to URLLoader, as it also encapsulates communication with NetworkService and browser process to deal with auth challenges, etc. It also indirectly owns an URLRequest (through WebSocketChannel => WebSocketStream => WebSocketStreamRequest. As can be seen in the class diagram above), though it currently does not handles client certificate requests coming through URLLoaderNetworkServiceObserver. That's that gap this proposal aims to address.

### Code diving

- [x] Is [ClientCertStoreNSS](https://source.chromium.org/chromium/chromium/src/+/main:net/ssl/client_cert_store_nss.h;l=23;drc=70b0ee470bd53a63da6c4194579e8c2865db2b79) the [net::ClientCertStore](https://source.chromium.org/chromium/chromium/src/+/main:net/ssl/client_cert_store.h;l=21;drc=70b0ee470bd53a63da6c4194579e8c2865db2b79) used on Linux Desktop?
  - Answer: Yes. In Chrome, created at [ProfileNetworkContextService::CreateClientCertStore](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/net/profile_network_context_service.cc;l=622-625;drc=169c6cc102b39295a5bfe2f2a176b42b1c2fe2c4) function.
- [x] Which [network::mojom::URLLoaderNetworkServiceObserver](https://source.chromium.org/chromium/chromium/src/+/main:services/network/public/mojom/url_loader_network_service_observer.mojom;l=83;drc=169c6cc102b39295a5bfe2f2a176b42b1c2fe2c4) impl is used for HTTPS?
  - Answer: [content::StoragePartitionImpl](https://source.chromium.org/chromium/chromium/src/+/main:content/browser/storage_partition_impl.cc;l=1853;drc=70b0ee470bd53a63da6c4194579e8c2865db2b79).
  - Steps to hit the relevant code paths:
    1. Install `badssl.com-client.p12` cert from https://badssl.com/download:
    ```sh=bash
    # Install it (tested on Arch Linux)
    pk12util -i ~/Downloads/badssl.com-client.p12 -d sql:$HOME/.pki/nssdb

    # To list all the certificates installed:
    certutil -d sql:$HOME/.pki/nssdb -L
    ```
    2. Load it in chrome:
    ```sh=bash
    out/linux/chrome \
      --enable-logging=stderr \
      --no-sandbox \
      --user-data-dir=/tmp/chr_tmp_linux \
      --vmodule='net/ssl/*=10,wayland*=10' \
      --ozone-platform=wayland \
      https://client.badssl.com
    ```
    3. This is the stack trace of its construction call (at browser process):
    ```c++
    [1201823:1201823:0622/203057.913491:ERROR:profile_network_context_service.cc(619)] ### Creating ClientCertStoreNSS instance.
    #0 0x7f0392475689 base::debug::CollectStackTrace()
    #1 0x7f0392361733 base::debug::StackTrace::StackTrace()
    #2 0x55896a03202d ProfileNetworkContextService::CreateClientCertStore()
    #3 0x7f038f6cbefe content::StoragePartitionImpl::OnCertificateRequested()
    #4 0x7f038ecbbd5b network::mojom::URLLoaderNetworkServiceObserverStubDispatch::Accept()
    #5 0x7f0391c72365 mojo::InterfaceEndpointClient::HandleValidatedMessage()
    ...
    #15 0x7f0391c17db0 mojo::SimpleWatcher::OnHandleReady()
    #16 0x7f0391c182d2 base::internal::Invoker<>::RunOnce()
    #17 0x7f03923fe769 base::TaskAnnotator::RunTaskImpl()
    #18 0x7f039242675a base::sequence_manager::internal::ThreadControllerWithMessagePumpImpl::DoWorkImpl()
    #19 0x7f0392425c8e base::sequence_manager::internal::ThreadControllerWithMessagePumpImpl::DoWork()
    #20 0x7f0392426f42 base::sequence_manager::internal::ThreadControllerWithMessagePumpImpl::DoWork()
    #21 0x7f0392390c83 base::MessagePumpGlib::Run()
    #22 0x7f0392427414 base::sequence_manager::internal::ThreadControllerWithMessagePumpImpl::Run()
    #23 0x7f03923cbb90 base::RunLoop::Run()
    #24 0x7f038f02d235 content::BrowserMainLoop::RunMainMessageLoop()
    #25 0x7f038f02ee92 content::BrowserMainRunnerImpl::Run()
    #26 0x7f038f02a6fb content::BrowserMain()
    #27 0x7f038fc46d15 content::RunBrowserProcessMain()
    #28 0x7f038fc486bb content::ContentMainRunnerImpl::RunBrowser()
    #29 0x7f038fc48142 content::ContentMainRunnerImpl::Run()
    #30 0x7f038fc45627 content::RunContentProcess()
    #31 0x7f038fc45702 content::ContentMain()
    #32 0x5589691a7533 ChromeMain
    #33 0x7f037f5fb310 __libc_start_call_main
    #34 0x7f037f5fb3c1 __libc_start_main_alias_2
    #35 0x5589691a732a _start

    ```
- [x] Where is the WebSocketStream created at NetworkService process side?
  - Answer: The entry point is [network::mojom::NetworkContext::CreateWebSocket()](https://source.chromium.org/chromium/chromium/src/+/main:services/network/public/mojom/network_context.mojom;l=1250;drc=da2ec58f64a4986e99d1799b711dade577a2a5f0), the same toplevel component/API used to create, for example, network::URLLoaderFactory instances (used to create URLLoaders all over the code base).
  - An example of its usage is ...
- [x] Which process/thread [WebSocketStream::Delegate::OnCertificateRequested()](https://source.chromium.org/chromium/chromium/src/+/main:net/websockets/websocket_stream.cc;l=426;drc=0e45c020c43b1a9f6d2870ff7f92b30a2f03a458) runs in?
  - Answer: It runs in NetworkService process, in the io thread.
  - Here is a sample stack trace of [net::WebSocketStreamRequestImpl::PerformUpgrade()](https://source.chromium.org/chromium/chromium/src/+/main:net/websockets/websocket_stream.cc;l=173;drc=0e45c020c43b1a9f6d2870ff7f92b30a2f03a458):
      ```c++
      #2 0x7fe46e6b6194 net::(anonymous namespace)::WebSocketStreamRequestImpl::PerformUpgrade()
      #3 0x7fe46e6b5af7 net::(anonymous namespace)::Delegate::OnResponseStarted()
      #4 0x7fe46e62a691 net::URLRequest::NotifyResponseStarted()
      #5 0x7fe46e63b26d net::URLRequestJob::NotifyFinalHeadersReceived()
      #6 0x7fe46e63aa3d net::URLRequestJob::NotifyHeadersComplete()
      #7 0x7fe46e633260 net::URLRequestHttpJob::NotifyHeadersComplete()
      #8 0x7fe46e635c55 net::URLRequestHttpJob::SaveCookiesAndNotifyHeadersComplete()
      #9 0x7fe46e6343ee net::URLRequestHttpJob::OnStartCompleted()
      #10 0x7fe46e4de32c net::HttpCache::Transaction::DoLoop()
      #11 0x7fe46e3e34fa base::internal::Invoker<>::RunOnce()
      #12 0x7fe46e4fbde0 net::HttpNetworkTransaction::DoCallback()
      #13 0x7fe46e69c305 net::WebSocketBasicHandshakeStream::ReadResponseHeadersCallback()
      #14 0x7fe46e4d23ad base::internal::Invoker<>::RunOnce()
      #15 0x7fe46e530923 net::HttpStreamParser::OnIOComplete()
      #16 0x7fe46e3e34fa base::internal::Invoker<>::RunOnce()
      #17 0x7fe46e5cd078 net::TCPClientSocket::DidCompleteRead()
      #18 0x7fe46e6770df net::TCPSocketPosix::ReadCompleted()
      #19 0x7fe46e67887a base::internal::Invoker<>::RunOnce()
      #20 0x7fe46e674e52 net::SocketPosix::RetryRead()
      #21 0x7fe46e675b15 net::SocketPosix::ReadCompleted()
      #22 0x7fe46e675904 net::SocketPosix::OnFileCanReadWithoutBlocking()
      #23 0x7fe47045e6c9 base::MessagePumpLibevent::OnLibeventNotification()
      #24 0x7fe47052be6c event_base_loop
      #25 0x7fe47045e9b1 base::MessagePumpLibevent::Run()
      #26 0x7fe4703c6414 base::sequence_manager::internal::ThreadControllerWithMessagePumpImpl::Run()
      #27 0x7fe47036ab90 base::RunLoop::Run()
      #28 0x7fe4703f5e6c base::Thread::Run()
      #29 0x7fe46cb14580 content::(anonymous namespace)::ChildIOThread::Run()
      #30 0x7fe4703f61ee base::Thread::ThreadMain()
      #31 0x7fe47042e51b base::(anonymous namespace)::ThreadFunc()
      #32 0x7fe45d5fa5c2 start_thread
      #33 0x7fe45d67f584 __GI___clone
      ```
- [x] Where in Chrome code does the certificate selection UI is shown?
  - At [ChromeContentBrowserClient::SelectClientCertificate()](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/chrome_content_browser_client.cc;l=3192;drc=da2ec58f64a4986e99d1799b711dade577a2a5f0) function, one of the toplevel content embedding class in Chrome code.
- [x] Where in the codebase the 'AutoSelectCertificateForUrls' policy is applied when selection a client certificate?
  - At [chrome::enterprise_util:: GetCertAutoSelectionFilters()](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/enterprise/util/managed_browser_utils.cc;l=63;drc=da2ec58f64a4986e99d1799b711dade577a2a5f0) function, called by the above mentioned [ChromeContentBrowserClient::SelectClientCertificate()](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/chrome_content_browser_client.cc;l=3192;drc=da2ec58f64a4986e99d1799b711dade577a2a5f0). Thus in //chrome layer (ie: higher level than //net for example, where core websocket code lives), which would make it difficult to access (ie: hacky, layering violation) in a quick&dirty workaround (as in the bullet below).
- [ ] ~~Would it be possible manipulate [net::ClientCertStore]() (same used in URLLoader path, though in browser process instead) straight away from [net::WebSocketStream]() code? That would simplify things in case of a more short-term approach(/workaround), eg: No mojo to communicate with browser process, no migration to URLLoader, etc.~~
  - Potential problem: Even not showing the Certificate Selection popup, it seems another dialog seems to be required in order to let users enter their private key password which, in Chrome, is implemented by [ChromeNSSCryptoModuleDelegate](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ui/crypto_module_delegate_nss.cc;l=73;drc=da2ec58f64a4986e99d1799b711dade577a2a5f0) and set up when [instantiating the net::ClientCertStore](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/net/profile_network_context_service.cc;l=622-625;drc=169c6cc102b39295a5bfe2f2a176b42b1c2fe2c4). This must be considered in our solution, it would be a potential blocker for supporting AutoSelectCertificateForUrls + WSS.
  - FTR, there is a [//remoting](https://source.chromium.org/chromium/chromium/src/+/main:remoting/host/token_validator_base.cc;l=205-236;drc=da2ec58f64a4986e99d1799b711dade577a2a5f0) component that does something similar, though it probably runs on its UI process, etc.

### Solution (WIP)

[network::WebSocket](https://source.chromium.org/chromium/chromium/src/+/main:services/network/websocket.h;drc=da2ec58f64a4986e99d1799b711dade577a2a5f0) class already receives (and stores) a [URLLoaderNetworkServiceObserver](https://source.chromium.org/chromium/chromium/src/+/main:services/network/public/mojom/url_loader_network_service_observer.mojom;l=83;drc=169c6cc102b39295a5bfe2f2a176b42b1c2fe2c4), though it is used only to forward [OnSSLCertificateError](https://source.chromium.org/chromium/chromium/src/+/main:services/network/websocket.cc;l=359-378;drc=da2ec58f64a4986e99d1799b711dade577a2a5f0) event as of now. So, the solution would consist of:

1. [x] Plumb WebSocketStream's [Delegate::OnCertificateRequest()](https://source.chromium.org/chromium/chromium/src/+/main:net/websockets/websocket_stream.cc;l=426;drc=da2ec58f64a4986e99d1799b711dade577a2a5f0) all the way up to [network::WebSocket](https://source.chromium.org/chromium/chromium/src/+/main:services/network/websocket.h;l=54;drc=da2ec58f64a4986e99d1799b711dade577a2a5f0) so that it can forward the request to the browser process, through its mojom::URLLoaderNetworkServiceObserver instance. Which requires to:
  - Add a new method to [net::WebSocketStream::ConnectDelegate](https://source.chromium.org/chromium/chromium/src/+/main:net/websockets/websocket_stream.h;l=91;drc=da2ec58f64a4986e99d1799b711dade577a2a5f0) interface and impl it in [net::WebSocketChannel](https://source.chromium.org/chromium/chromium/src/+/main:net/websockets/websocket_channel.h;l=51;drc=da2ec58f64a4986e99d1799b711dade577a2a5f0)
  - Add a new method to [net::WebSocketChannel::WebSocketEventInterface](https://source.chromium.org/chromium/chromium/src/+/main:net/websockets/websocket_event_interface.h;l=35;drc=da2ec58f64a4986e99d1799b711dade577a2a5f0) and impl it in [network::WebSocket::WebSocketEventHandler](https://source.chromium.org/chromium/chromium/src/+/main:services/network/websocket.cc;l=132;drc=da2ec58f64a4986e99d1799b711dade577a2a5f0) which builds required data and calls into [URLLoaderNetworkServiceObserver::OnCertificateRequested()](https://source.chromium.org/chromium/chromium/src/+/main:services/network/public/mojom/url_loader_network_service_observer.mojom;l=104-107;drc=da2ec58f64a4986e99d1799b711dade577a2a5f0). [URLLoader's impl](https://source.chromium.org/chromium/chromium/src/+/main:services/network/url_loader.cc;l=1489-1505;drc=da2ec58f64a4986e99d1799b711dade577a2a5f0) should be a good ref.

2. [x] Patch network::WebSocket to implement [network::mojom::ClientCertificateResponder](https://source.chromium.org/chromium/chromium/src/+/main:services/network/public/mojom/url_loader_network_service_observer.mojom;l=30;drc=da2ec58f64a4986e99d1799b711dade577a2a5f0), just like URLLoader does, thus forwarding the certificate selection result back to the URLRequest, owned by the corresponding net::WebSocketStream. Which could be achieved by:
  - Add boilerplate code that handles a new mojo::Receiver for mojom::ClientCertificateResponder, etc.
  - Add an implementation of [net::SSLPrivateKey](https://source.chromium.org/chromium/chromium/src/+/main:net/ssl/ssl_private_key.h;drc=da2ec58f64a4986e99d1799b711dade577a2a5f0;l=29), similar to [URLLoader's one](https://source.chromium.org/chromium/chromium/src/+/main:services/network/url_loader.cc;l=217;drc=da2ec58f64a4986e99d1799b711dade577a2a5f0A). _(TODO: Really needed? learn more about it.)_
  - Use [URLLoader's impl](https://source.chromium.org/chromium/chromium/src/+/main:services/network/url_loader.cc;l=2036-2046;drc=da2ec58f64a4986e99d1799b711dade577a2a5f0) as reference.

3. [ ] Modify (slightly?) Chrome's impl of [ChromeContentBrowserClient::SelectClientCertificate()](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/chrome_content_browser_client.cc;l=3192;drc=da2ec58f64a4986e99d1799b711dade577a2a5f0) such that it avoids the Cert Selection UI popup, when it corresponds to a websocket request (how?). Likely optional for the initial short-term solution, though a hard-requirement for the final upstreamable solution.
  - WIP: under investigation.

**WIP Patch at:** https://chromium-review.googlesource.com/c/chromium/src/+/3733743

### Testing

- I've set up a test nginx server at `wss://yamane.dev:443`, which consists of:
  - A deploy of [this PHP websocket demo project](https://github.com/chesslablab/chess-server);
  - Behind an nginx proxy which adds a SSL layer, configured with a letsencrypt certificate and client auth enabled (using a self-signed client cert, as described [here](https://pavelevstigneev.medium.com/setting-nginx-with-letsencrypt-and-client-ssl-certificates-3ae608bb0e66))
  - In order to check if your certificate/key pair properly authenticates against the above server, you can run, for example:
  ```sh=bash
  openssl s_client -connect yamane.dev:443 -cert path/to/client.crt -key path/to/client.key
  ```
- Ensure 'AutoSelectCertificateForUrls' policy is in place. By putting, for example, [auto-select-cert-policy.json](https://github.com/nickdiego/chromium-docs/blob/main/wss-client-certs/auto-select-cert-policy.json) file into `/etc/chromium/policies/managed` directory (note: it may vary depending on how chromium is installed on your distro/devel env). Its content:
```json
{
  "AutoSelectCertificateForUrls": [
    "{ \"pattern\": \"yamane.dev\", \"filter\": { \"ISSUER\": { \"CN\": \"Nick Yamane\" } } }"
  ]
}
```
- Run patched chromium as follows:
```sh=bash
out/linux/chrome \
  --ozone-platform=wayland \
  --enable-logging=stderr \
  --no-sandbox \
  --user-data-dir=/tmp/chr \
  --vmodule='websocket*=1,*nss*=5,*ssl*=5,wayland*=5' \
  --auto-open-devtools-for-tabs
```
- Assuming you have the client cert installed in the platform certificate store (NSS), run the following code in Chromium's developer console:
```js
// If no error shows up after running this, it means the client cert (auto)
// selection + handshake/connection succeeded.
const ws = new WebSocket('wss://yamane.dev:443');

// So run this, for example, to ensure we can talk to the wss server.
ws.onmessage = (res) => { console.log(res.data) };
ws.send('/start analysis');
```

#### WS sample server troubleshooting (FTR):
  - Had to create an empty `data/players.json` (eg: `echo '[]' > data/players.json`) file otherwise the server was failing to start with the following error:
```
PHP Warning:  file_get_contents(src/../data/players.json): Failed to open stream: No such file or directory in vendor/chesslablab/php-chess/src/Grandmaster.php on line 13
PHP Fatal error:  Uncaught TypeError: ArrayIterator::__construct(): Argument #1 ($array) must be of type array, null given in vendor/chesslablab/php-chess/src/Grandmaster.php:16
```

### Some links:
- https://textslashplain.com/2020/05/04/client-certificate-authentication/
- https://jpassing.com/2021/09/27/do-browsers-use-client-certificates-to-authenticate-the-user-the-device-or-both/
- http://www.browserauth.net/tls-client-authentication
- About AutoSelectCertificateForUrls policy:
  - https://groups.google.com/a/chromium.org/g/chromium-discuss/c/oBuvub4m_zs
  - https://chromeenterprise.google/policies/#AutoSelectCertificateForUrls
  - https://gist.github.com/IngussNeilands/3bbbb7d78954c85e2e988cf3bfec7caa
  - https://chromium.googlesource.com/chromium/src/+/HEAD/docs/enterprise/policies.md
  - https://www.chromium.org/administrators/linux-quick-start/
- https://wiki.archlinux.org/title/Network_Security_Services
- https://firefox-source-docs.mozilla.org/security/nss/legacy/tools/nss_tools_certutil/index.html#Using_the_Certificate_Database_Tool
- https://firefox-source-docs.mozilla.org/security/nss/legacy/reference/nss_certificate_functions/index.html#cert_getdefaultcertdb
- https://man.archlinux.org/man/core/nss/pk12util.1.en
- https://chromium.googlesource.com/chromium/src/+/refs/heads/master/net/docs/life-of-a-url-request.md
- https://medium.com/geekculture/creating-a-local-websocket-server-with-tls-ssl-is-easy-as-pie-de1a2ef058e0
- https://medium.com/geekculture/a-simple-example-of-ssl-tls-websocket-with-reactphp-and-ratchet-e03be973f521
- https://help.zerossl.com/hc/en-us/articles/360058296034-Certificate-Format
- https://www.electricmonk.nl/log/2018/06/02/ssl-tls-client-certificate-verification-with-python-v3-4-sslcontext/
- https://www.nginx.com/blog/using-free-ssltls-certificates-from-lets-encrypt-with-nginx/
- https://www.digitalocean.com/community/tutorials/how-to-set-up-let-s-encrypt-with-nginx-server-blocks-on-ubuntu-16-04
- https://pavelevstigneev.medium.com/setting-nginx-with-letsencrypt-and-client-ssl-certificates-3ae608bb0e66
