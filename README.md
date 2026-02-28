# htb-challenge-apkey
An HackTheBox challenge of Android .apk that demonstrate the decompiling of the app in order to find an unique key hidden somewhere

---
## Challenge Scenario
```
This app contains some unique keys. Can you get one?
```

Basically, an APK file is an archiev, so to decompile, we need to extract the package and find the `.dex` file
```
└─$ unzip ./assets/apk/orig/APKey.apk -d ./assets/apk/modif/decompiled 
```
```
└─$ find ./assets/apk/modif/decompiled -type f -iname '*.dex'             
./assets/apk/modif/decompiled/classes.dex
```
```
└─$ find ./assets/apk/modif/decompiled -type f -iname '*.dex'|xargs -n1 -I{} /usr/bin/jadx -d $(pwd)/assets/apk/modif/out-jadx-$RANDOM $(pwd)/assets/apk/modif/decompiled/classes.dex 2>/dev/null
INFO  - loading ...
INFO  - processing ...
ERROR - finished with errors, count: 2
```

