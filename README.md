# Demo cordova project for plugin installation bug

## Steps to reproduce

After cloning the repo `cd` into the root directory of the project and run:

* `npm install`
* `cordova prepare ios --verbose`

The plugins will fail to install correctly and the contents of the generated `plugins/fetch.json` file is messed up:

```json
{
  "cordova-plugin-whitelist": {
    "source": {
      "type": "registry",
      "id": "cordova-plugin-whitelist@1.3.1"
    },
    "is_top_level": true,
    "variables": {}
  },
  "cordova-plugin-splashscreen": {
    "source": {
      "type": "registry",
      "id": "cordova-plugin-customurlscheme@^4.3.0"
    },
    "is_top_level": true,
    "variables": {
      "URL_SCHEME": "io.cordova.hellocordova"
    }
  }
}
```

Also, the output of the `cordova prepare ios --verbose` makes it quite clear that something went wrong during the installation:

```
...
Discovered plugin "cordova-plugin-customurlscheme" in config.xml. Adding it to the project
No scripts found for hook "before_plugin_add".
Calling plugman.fetch on plugin "cordova-plugin-customurlscheme@^4.3.0"
Running command: npm install cordova-plugin-customurlscheme@^4.3.0 --production --no-save
Command finished with error code 0: npm install,cordova-plugin-customurlscheme@^4.3.0,--production,--no-save
Copying plugin "/path-to-project/cordova-install-plugin-bug/node_modules/cordova-plugin-splashscreen" => "/path-to-project/cordova-install-plugin-bug/plugins/cordova-plugin-splashscreen"
Calling plugman.install on plugin "/path-to-project/cordova-install-plugin-bug/plugins/cordova-plugin-splashscreen" for platform "ios
Plugin "cordova-plugin-splashscreen" already installed on ios.
...
```

The installation process seems to mess up the installation of the `customurlscheme` plugin with that of the `splashscreen` plugin.

## Underlying code issue

The root of the problem seems to be in the [add.js](https://github.com/apache/cordova-lib/blob/81f0a8214aa6dc2811bcb6472f4ce62cb8719024/src/cordova/plugin/add.js) `cordova-lib` file.

[Line 97](https://github.com/apache/cordova-lib/blob/81f0a8214aa6dc2811bcb6472f4ce62cb8719024/src/cordova/plugin/add.js#L97) prints the `Calling plugman.fetch on plugin "customurlscheme"` line, while [line 131](https://github.com/apache/cordova-lib/blob/81f0a8214aa6dc2811bcb6472f4ce62cb8719024/src/cordova/plugin/add.js#L131) prints `Calling plugman.install on plugin "splashscreen"`.

This seems to suggest that somewhere between these two lines cordova messes up the plugins somehow.

The code is somewhat difficult to understand but the root cause seems to be this [pluginInfo](https://github.com/apache/cordova-lib/blob/81f0a8214aa6dc2811bcb6472f4ce62cb8719024/src/cordova/plugin/add.js#L52) global variable. Its value is [assigned in the callback of a promise](https://github.com/apache/cordova-lib/blob/81f0a8214aa6dc2811bcb6472f4ce62cb8719024/src/cordova/plugin/add.js#L103), read from [another callback of a promise](https://github.com/apache/cordova-lib/blob/81f0a8214aa6dc2811bcb6472f4ce62cb8719024/src/cordova/plugin/add.js#L131), while [a whole chain of promises](https://github.com/apache/cordova-lib/blob/81f0a8214aa6dc2811bcb6472f4ce62cb8719024/src/cordova/plugin/add.js#L74) is set using a `reduce` function.

## Further information

Replacing `"cordova-plugin-splashscreen": "^5.0.1"` with `"cordova-plugin-splashscreen": "5.0.1"` in `package.json` makes the `prepare` command complete succesfully ([demo branch](https://github.com/vially/cordova-plugin-install-bug-demo/tree/no-fail)). This is a small example of the volatility of the bug.

## Testing Environment

* Node `9.5.0`
* NPM `5.6.0`
* Cordova `8.0.0`
