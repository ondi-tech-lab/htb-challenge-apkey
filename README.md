# htb-challenge-apkey
An HackTheBox challenge of Android .apk that demonstrate the decompiling of the app in order to find an unique key hidden somewhere

---
## Challenge Scenario
```
This app contains some unique keys. Can you get one?
```
If we run the application, and enter any non valid credential, it will show a `toast` message `Wrong Credentials!`. Since this application size is quite small, my first guess was that this is a one form simple application.


Basically, an APK file is an archiev, so to decompile, we need to extract the package and find the `.dex` file
```
└─$ unzip ./assets/apk/orig/APKey.apk -d ./assets/apk/modif/decompiled 
```
```
└─$ find ./assets/apk/modif/decompiled -type f -iname '*.dex'             
./assets/apk/modif/decompiled/classes.dex
```

```
└─$ find ./assets/apk/modif/decompiled -type f -iname '*.dex'|xargs -n1 -I{} /usr/local/bin/jadx -d $(pwd)/assets/apk/modif/out-jadx-$RANDOM $(pwd)/assets/apk/modif/decompiled/classes.dex 2>
/dev/null                                                                                                                                                                                     
INFO  - loading ...                                                                            
INFO  - processing ...                                                                         
INFO  - doneress: 479 of 557 (85%) 
```

By looking through the extracted folder, we found the target file based on the `toast` message we found early
```
└─$ grep -Hnri 'wrong credentials' ./assets/apk/modif/out-jadx-21719 
./assets/apk/modif/out-jadx-21719/sources/com/example/apkey/MainActivity.java:92:                java.lang.String r1 = "Wrong Credentials!"
```

And there are few strings value that written on the `MainActivity.java` file as follow
```
└─$ grep -nEo '"+([a-zA-Z0-9\!\-_\ ]{1,})+"' ./assets/apk/modif/out-jadx-21719/sources/com/example/apkey/MainActivity.java
36:"admin"
41:"MD5"
53:"a2a3d412e92d896134d9c9126d756f"
62:"Wrong Credentials!"
```
And if we look at the full `MainActivity.java' file, its shows and encoding flow from `MD5` to `Hex` for `password input`  at the `onClickListener` event
```
        @Override // android.view.View.OnClickListener
        public void onClick(View view) {
            Toast makeText;
            String str;                       
            try {                                                                                                                                                                             
                if (MainActivity.this.f928c.getText().toString().equals("admin")) {
                    MainActivity mainActivity = MainActivity.this;
                    b bVar = mainActivity.e;
                    String obj = mainActivity.d.getText().toString();
                    try {
                        MessageDigest instance = MessageDigest.getInstance("MD5");
                        instance.update(obj.getBytes());
                        byte[] digest = instance.digest();
                        StringBuffer stringBuffer = new StringBuffer();
                        for (byte b2 : digest) {
                            stringBuffer.append(Integer.toHexString(b2 & 255));
                        }
                        str = stringBuffer.toString();
                    } catch (NoSuchAlgorithmException e) {
                        e.printStackTrace();
                        str = "";
                    }
                    if (str.equals("a2a3d412e92d896134d9c9126d756f")) {
                        Context applicationContext = MainActivity.this.getApplicationContext(); 
                        MainActivity mainActivity2 = MainActivity.this;
                        b bVar2 = mainActivity2.e;
                        g gVar = mainActivity2.f;
                        makeText = Toast.makeText(applicationContext, b.a(g.a()), 1);
                        makeText.show();
                    }
                }
                makeText = Toast.makeText(MainActivity.this.getApplicationContext(), "Wrong Credentials!", 0);
                makeText.show();
            } catch (Exception e2) {
                e2.printStackTrace();
            }
        }
``` 

## Bypass
There are two possible way that we can do to bypass this, which are
- Bypass the program flow by Frida (or any script injection available, I choose this one for familiarity)
- Modify and rebuild the apk package by replace the harcoded hash password string that we found
- Crack the hardcoded string with tools like Hashcat (This is a hard way, and we are not gonna get into that)

I'll go with the first two, since I have tried crack the string several times but didn't make it.


-[v] Bypass program process/ Spawn Android process with Frida
As I mentioned, we may collect all the string called in the app process and compare the value with hash string we got from the reversed apk file, since this is a simple app, that's would be okay to do it forcefully on all strings called on its process, and here is the final javascipt file to bypass the login process.
***bypass.js***
```
Java.perform(function () {

    var StringClass = Java.use("java.lang.String");

    var originalEquals = StringClass.equals.overload("java.lang.Object");

    StringClass.equals.overload("java.lang.Object").implementation = function (obj) {

        // prevent null pointer
        if (obj) {

            var comparedValue = obj.toString();

            // Target hash of the APK
            if (comparedValue === "a2a3d412e92d896134d9c9126d756f") {

                console.log("[+] Bypassing password hash comparison!");
                return true;
            }
        }

        // Default behaviour
        return originalEquals.call(this, obj);
    };

});
```
