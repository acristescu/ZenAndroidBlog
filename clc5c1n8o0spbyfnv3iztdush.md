# Adding the Support Library through Maven

![Adding the Support Library through Maven](https://cdn.hashnode.com/res/hashnode/image/upload/v1672091392910/HqOPcPaiV.png)

In case you've missed it, at the last Google IO, Google announced that you can now add the Support Library directly from gradle without having to download the "support repository" anymore. This should come as good news to those of us who have to keep a Jenkins/Travis CI system up and running. Neat!

Basically, all you have to do is make sure your main `build.gradle` file contains a `maven` section and that `https://maven.google.com` is specified as an endpoint:

    allprojects {  
        repositories {
            jcenter()
            maven {
                url "https://maven.google.com"
            }
        }
    }
    

Then, you can add your support library as a dependency like you would any other library:

    'com.android.support:appcompat-v7:25.3.1'  
    

No need to download the magic package called `Support Repository` through the SDK manager and definitely no need to keep it up to date. While this was annoying to do on the development machines, it was an even greater pain to maintain on the Continuous Integration machines, where you would generally have to use VNC to connect and update them manually.