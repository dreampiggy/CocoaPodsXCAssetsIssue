# CocoaPods Bug


# What's for

It seems that CocoaPods have a really huge bug about Asset Catalog, the root case came from 4 years ago, but not been fixed till today. (The latest 1.6.0-beta2 still have this issue). I create a demo to trigger the issue and show the result.

# Issue

CocoaPods support you to use Asset Catalog `xcassets` inside Pods. You can use one of these two syntax to include a xcassets resource.

+ Using `resource_bundles`

```
s.resource_bundles = {
'TestLibrary' => ['TestLibrary/Assets/TestLibrary.xcassets']
}
```

+ Using `resources`

```
s.resources = ['TestLibrary2/Assets/TestLibrary2.xcassets']
```

However, when your Application project, including at least one Pods using the `resources` with xcassets. Then all the Pods' xcassets resource (whatever it come from), will be duplicated twice, inside your App's main bundle. It stored inside your App's product app's root path via `Assets.car`.

See the screenshot here. The App's main bundle `Assets.car`, contains all the other Pods's image resources. However, the `TestLibrary` itself also contains its image resources.

![](./Bug.png)

But, since `TestLibrary` using `resource_bundles`, its resource should not appear outside its bundle. The Main bundle should not contains 1-5.jpg.

# Reproduce

Clone this repo, run `pod install`, then build and check the Product app.

You can use [cartool](https://github.com/steventroughtonsmith/cartool) to export images inside `Assets.car`

# Fix or workaround ?

The bug was cause by the shell script phase. `[CP] Copy Pods Resources`. To fix that, just modify one line inside that script. The script is generated each time you run `pod install`.

Original:

```bash
  # Find all other xcassets (this unfortunately includes those of path pods and other targets).
  OTHER_XCASSETS=$(find "$PWD" -iname "*.xcassets" -type d)
  while read line; do
    if [[ $line != "${PODS_ROOT}*" ]]; then
      XCASSET_FILES+=("$line")
    fi
  done <<<"$OTHER_XCASSETS"
```

Patch:

```diff
<     if [[ $line != "${PODS_ROOT}*" ]]; then
---
>     if [[ $line != ${PODS_ROOT}* ]]; then
```

It seems that the author want to do filter of that `Pods` folder, but use the wrong shell syntax. The condition check always passed.

For workaround, add this patch in your `Podfile`'s post_install script. Because you can not always assume all Pods using xcassets with `resources`. If at least 1 use that syntax, you'll enter the trap :)

