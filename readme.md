# pin-proxy-for-container
This extension allows HTTP, HTTPS, SOCKS4, and SOCKS5 proxies to be assigned to individual containers. Proxy settings are saved automatically and can be temporarily enabled or disabled from the main container list.

## Installation

- [addons.mozilla.org](https://addons.mozilla.org/firefox/addon/pin-proxy-for-container)

### Building

- To build and package the extension for distribution, run:
    ```bash
    yarn install
    yarn build
    ```
    - This will run the webpack build and place the output in `build/webpack`, after which the extension will be packaged using `web-ext` and the output will be placed in `web-ext-artifacts`.
