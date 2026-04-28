pnpm add @tarojs/plugin-platform-weapp
 WARN  deprecated @types/classnames@2.3.4: This is a stub types definition. classnames provides its own type definitions, so you do not need this installed.
 WARN  deprecated @react-native-community/cameraroll@4.1.2: Package has been moved to @react-native-camera-roll/camera-roll starting with version 5.0
 WARN  deprecated react-native-image-resizer@1.4.5: 🚨 react-native-image-resizer has moved to @bam.tech/react-native-image-resizer
 WARN  deprecated react-native-unimodules@0.11.0: replaced by the 'expo' package, learn more: https://blog.expo.dev/whats-new-in-expo-modules-infrastructure-7a7cdda81ebc
 WARN  deprecated @react-native-community/picker@1.8.1: this library has been renamed to @react-native-picker/picker. See the GitHub new repository: https://github.com/react-native-picker/picker
 WARN  deprecated @react-native-community/viewpager@4.2.4: This library is no longer supported. Please use react-native-pager-view instead
 WARN  deprecated @react-native-community/masked-view@0.1.11: Repository was moved to @react-native-masked-view/masked-view
 WARN  deprecated @react-native-community/async-storage@1.12.1: Async Storage has moved to new organization: https://github.com/react-native-async-storage/async-storage
 WARN  deprecated react-native-web@0.13.18: < 0.16.0 is no longer supported
 WARN  deprecated @unimodules/core@5.1.2: replaced by the 'expo' package, learn more: https://blog.expo.dev/whats-new-in-expo-modules-infrastructure-7a7cdda81ebc
 WARN  51 deprecated subdependencies found: @babel/plugin-proposal-class-properties@7.10.4, @babel/plugin-proposal-class-properties@7.12.1, @babel/plugin-proposal-nullish-coalescing-operator@7.18.6, @babel/plugin-proposal-object-rest-spread@7.11.0, @babel/plugin-proposal-object-rest-spread@7.20.7, @babel/plugin-proposal-optional-catch-binding@7.18.6, @babel/plugin-proposal-optional-chaining@7.21.0, @hapi/address@4.1.0, @hapi/formula@2.0.0, @hapi/joi@17.1.1, @npmcli/move-file@1.1.2, @react-native-community/bob@0.16.2, @stylelint/postcss-css-in-js@0.37.3, @stylelint/postcss-markdown@0.36.2, @swc/register@0.1.10, @unimodules/core@5.5.1, @unimodules/react-native-adapter@5.6.0, abab@2.0.6, babel-eslint@10.1.0, babel-eslint@8.2.6, babel-plugin-remove-dead-code@1.3.2, chokidar@2.1.8, circular-json@0.3.3, copy-concurrently@1.0.5, core-js@2.6.12, cross-spawn-async@2.2.5, debug@4.1.1, deep-assign@3.0.0, domexception@1.0.1, figgy-pudding@3.5.2, fs-write-stream-atomic@1.0.10, fsevents@1.2.13, fsevents@2.1.3, har-validator@5.1.5, html-webpack-plugin@3.2.0, inflight@1.0.6, move-concurrently@1.0.1, querystring@0.2.1, react-native-swipeout@2.3.6, request-promise-native@1.0.9, request@2.88.2, resolve-url@0.2.1, source-map-resolve@0.5.3, source-map-url@0.4.1, trim@0.0.1, tslint@6.1.3, urix@0.1.0, uuid@3.4.0, vm2@3.9.19, w3c-hr-time@1.0.2, webpack-chain@4.9.0
Packages: +3 -4
+++----
Progress: resolved 2807, reused 2765, downloaded 1, added 3, done

dependencies:

- @tarojs/plugin-platform-weapp 4.2.0

 WARN  Issues with peer dependencies found
.
├─┬ @tarojs/taro-rn 4.2.0
│ ├── ✕ unmet peer react@^18: found 17.0.2
│ ├── ✕ unmet peer react-native@^0.73.1: found 0.85.2
│ ├── ✕ unmet peer react-native-safe-area-context@4.8.2: found 3.4.1
│ ├── ✕ unmet peer expo@~50.0.0: found 55.0.17
│ ├── ✕ unmet peer expo-av@~13.10.6: found 8.6.0
│ ├── ✕ unmet peer expo-camera@~14.1.3: found 9.0.0
│ ├── ✕ unmet peer @react-native-async-storage/async-storage@1.21.0: found 1.24.0
│ ├── ✕ unmet peer @react-native-community/geolocation@^3.4.0: found 2.1.0
│ ├── ✕ unmet peer @react-native-community/netinfo@11.1.0: found 5.9.10
│ ├── ✕ unmet peer expo-barcode-scanner@~12.9.3: found 9.0.0
│ ├── ✕ unmet peer expo-brightness@~11.8.0: found 8.4.0
│ ├── ✕ unmet peer expo-file-system@~16.0.9: found 9.2.0
│ ├── ✕ unmet peer expo-image-picker@~14.7.1: found 9.1.1
│ ├── ✕ unmet peer expo-keep-awake@~12.8.2: found 8.4.0
│ ├── ✕ unmet peer expo-location@~16.5.5: found 9.0.1
│ ├── ✕ unmet peer expo-sensors@~12.9.1: found 9.2.0
│ ├─┬ @tarojs/runtime-rn 4.2.0
│ │ ├── ✕ unmet peer react@">= 18": found 17.0.2
│ │ └─┬ @tarojs/components-rn 4.2.0
│ │ ├── ✕ missing peer @react-native-picker/picker@2.6.1
│ │ ├── ✕ unmet peer react@^18: found 17.0.2
│ │ ├── ✕ unmet peer react-native@^0.73.1: found 0.85.2
│ │ ├── ✕ unmet peer expo@~50.0.0: found 55.0.17
│ │ ├── ✕ unmet peer expo-av@~13.10.6: found 8.6.0
│ │ ├── ✕ unmet peer expo-camera@~14.1.3: found 9.0.0
│ │ ├── ✕ unmet peer react-native-webview@13.6.4: found 10.10.2
│ │ ├─┬ @ant-design/react-native 5.0.0
│ │ │ └── ✕ missing peer @react-native-picker/picker@^1.9.10
│ │ ├─┬ @tarojs/router-rn 4.2.0
│ │ │ ├── ✕ unmet peer react@^18: found 17.0.2
│ │ │ ├── ✕ unmet peer react-native-screens@~3.29.0: found 2.18.1
│ │ │ ├── ✕ unmet peer react-native@^0.73.1: found 0.85.2
│ │ │ ├── ✕ unmet peer react-native-gesture-handler@~2.14.0: found 1.10.3
│ │ │ ├── ✕ unmet peer react-native-safe-area-context@4.8.2: found 3.4.1
│ │ │ ├─┬ @react-navigation/bottom-tabs 6.6.1
│ │ │ │ └── ✕ unmet peer react-native-screens@">= 3.0.0": found 2.18.1
│ │ │ ├─┬ @react-navigation/native-stack 6.11.0
│ │ │ │ └── ✕ unmet peer react-native-screens@">= 3.0.0": found 2.18.1
│ │ │ └─┬ @react-navigation/stack 6.4.1
│ │ │ └── ✕ unmet peer react-native-screens@">= 3.0.0": found 2.18.1
│ │ └─┬ @tarojs/components 4.2.0
│ │ └─┬ @tarojs/taro 4.2.0
│ │ ├── ✕ unmet peer @types/react@^18: found 16.14.69
│ │ ├── ✕ unmet peer postcss@^8: found 6.0.23 in @tarojs/cli
│ │ └── ✕ unmet peer webpack@^5: found 4.46.0 in @tarojs/mini-runner
│ └─┬ @tarojs/taro 4.2.0
│ ├── ✕ unmet peer @tarojs/components@4.2.0: found 3.2.10
│ ├── ✕ unmet peer @types/react@^18: found 16.14.69
│ ├── ✕ unmet peer postcss@^8: found 6.0.23 in @tarojs/cli
│ └── ✕ unmet peer webpack@^5: found 4.46.0 in @tarojs/mini-runner
├─┬ babel-preset-taro 3.2.10
│ └─┬ @tarojs/taro-rn 3.2.10
│ └─┬ @tarojs/runtime-rn 3.2.10
│ └─┬ @tarojs/components-rn 3.2.10
│ └── ✕ unmet peer @react-native-community/slider@4.0.0-rc.1: found 4.4.2
├─┬ eslint-plugin-react-hooks 1.7.0
│ └── ✕ unmet peer eslint@"^3.0.0 || ^4.0.0 || ^5.0.0 || ^6.0.0": found 7.32.0
├─┬ @taroify/core 0.0.29-alpha.8
│ ├── ✕ unmet peer @tarojs/taro@^3.3.9: found 3.2.10
│ ├── ✕ unmet peer @tarojs/components@^3.3.9: found 3.2.10
│ ├─┬ @taroify/hooks 0.0.29-alpha.9
│ │ └── ✕ unmet peer @tarojs/taro@^3.3.6: found 3.2.10
│ └─┬ @taroify/icons 0.0.29-alpha.9
│ └── ✕ unmet peer @tarojs/components@^3.3.9: found 3.2.10
├─┬ @tarojs/plugin-platform-h5 4.1.7
│ ├─┬ @tarojs/taro-h5 4.1.7
│ │ └─┬ @tarojs/router 4.1.7
│ │ └── ✕ unmet peer @tarojs/taro@4.1.7: found 3.2.10
│ └─┬ @tarojs/components-react 4.1.7
│ └─┬ @tarojs/components 4.1.7
│ └─┬ @tarojs/taro 4.1.7
│ ├── ✕ unmet peer @types/react@^18: found 16.14.69
│ ├── ✕ unmet peer postcss@^8: found 6.0.23 in @tarojs/cli
│ └── ✕ unmet peer webpack@^5: found 4.46.0 in @tarojs/mini-runner
├─┬ @tarojs/react 3.2.10
│ └─┬ react-reconciler 0.25.1
│ └── ✕ unmet peer react@^16.13.1: found 17.0.2
├─┬ expo-camera 9.0.0
│ └─┬ @koale/useworker 3.4.0
│ └── ✕ unmet peer react@^16.8.0: found 17.0.2
├─┬ react-native 0.85.2
│ ├── ✕ unmet peer react@^19.2.3: found 17.0.2
│ ├── ✕ unmet peer @types/react@^19.1.1: found 16.14.69
│ └─┬ @react-native/virtualized-lists 0.85.2
│ └── ✕ unmet peer @types/react@^19.2.0: found 16.14.69
├─┬ @react-native-community/picker 1.8.1
│ └── ✕ unmet peer react@^16.0: found 17.0.2
├─┬ @react-native-community/async-storage 1.12.1
│ └── ✕ unmet peer react@^16.8: found 17.0.2
├─┬ react-native-webview 10.10.2
│ └── ✕ unmet peer react-native@">=0.60 <0.64": found 0.85.2
└─┬ expo 55.0.17
└─┬ @expo/cli 55.0.26
└─┬ @expo/config 55.0.15
└─┬ @expo/require-utils 55.0.4
└── ✕ unmet peer typescript@"^5.0.0 || ^5.0.0-0": found 4.9.5
✕ Conflicting peer dependencies:
@react-native-picker/picker

Done in 41.2s
