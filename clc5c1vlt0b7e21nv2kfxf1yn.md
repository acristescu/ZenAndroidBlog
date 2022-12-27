# An evening of Firebase - event review

![An evening of Firebase - event review](https://cdn.hashnode.com/res/hashnode/image/upload/v1672091401872/h6w7T-jmh.png)

I attended Skill Matter's meetup/talk titled 'An evening of Firebase' and I wanted to share with you (and future me) my mental notes from what was presented.

The evening consisted of two talks: one held by Google's [Laurence Moroney](http://www.laurencemoroney.com/) and another by [Beeline](https://beeline.co) CTO Chetan Padia. The event was a one-off probably caused by Mr. Maroney fortuitous visit to London. The event was organised by `Skills Matter` and `Londroid: The London Android User Group`.

Firebase overview
-----------------

The first talk and the main event of the night was Laurence Moroney's `Firebase Overview`. While we all had to use some part of Firebase at one point or another in recent years, it was nice to get a holistic view of how Google envisions people using Firebase and not as a loose collection of libraries and services.

He started the talk by presenting some stats about how much people earn using their apps and trying to identify the habits of successful developers. While this would have been a very interesting topic for a whole talk, he focused on just one aspect: successful developers build cross platform apps: they target Android, iOS and Web at the same time.

He then continued by arguing that Firebase is meant to cater to that type of developer needs: powerful, quick-to-use services that are consistent across platforms.

The rest of the talk was a laundry list of the components that make up Firebase and how they help successful devs either quickly develop cross-platform apps or grow their user base. This slide was pretty much a table of contents:

![An evening of Firebase - event review](https://cdn.hashnode.com/res/hashnode/image/upload/v1672091403828/zMGVP0_FQ.png)

Here is a very short version of all these 16 sections:

1.  **Realtime Database** - scalable NoSQL database with realtime updates and no back-end code required. See my other articles on the subject.
2.  **Authentication** - drop-in solution for authenticating users. No more handling OAuth tokens yourself. Definitely worth a look.
3.  **Cloud Storage** - store and download files in the cloud.
4.  **Test Lab for Android** - test your app on hundreds of devices.
5.  **Crash reporting** - Yet another crash reporting solution. This one seems to be integrated with Analytics somehow.
6.  **Cloud functions** - The new kid on the block. Allows you to run pieces of (Javascript) code in response to database changes without having a proper backend in place.
7.  **Hosting** - Free hosting solution for apps
8.  **Performance Monitoring** - Ability to monitor performance parameters for your users. Integrated with Analytics, so you can see the latency by country for example
9.  **Google Analytics** - The flagship product, now encompasses the old Firebase analytics and Google Analytics
10.  **Dynamic Links** - Deep links within your app - not sure how useful these are really
11.  **Invites** - Make it easy for your users to share your app
12.  **AdMob** - Make money by showing ads in your app
13.  **Cloud Messaging** - push notifications
14.  **Remote Config** - push config values to users (in real time) based on the google analytics category they are in.
15.  **App indexing** - ability to make the phone google searches return content from your app if relevant (and your app is already installed)
16.  **AdWords** - pay google to promote your app basically.

Firebase at Beeline
-------------------

The second talk was a presentation by [Beeline](https://beeline.co) CTO Chetan Padia. He shortly introduced the Beeline startup and then proceeded to discuss how they used Firebase services to develop their product.

Beeline's product is a navigation gadget for cyclists. It's not a Garmin GPS though, it's more akin to an Android Watch, in that it connects through Bluetooth to your mobile phone (be it Android or iOS) and uses your phone's GPS and data connection to navigate and track your position. Furthermore, the gadget is not a true navigator in that it does not do turn-by-turn navigation, it simply points an arrow in the direction your intended destination. My personal opinion is that it's a nice concept, but I find the £100 price tag a tad much...

He then continued by describing the rationale behind them using Firebase and the difficulties they encountered by using it in production. He was really happy with the way it worked out, particularity the fact that they had almost no back-end code, however I did get the distinct impression that their system to test and ensure consistency and security of the database structure took a lot of time and effort to set up.

He demonstrated their system for generating and unit testing the `rules.json` file, which controls both the access rights and the structure of the database. He introduced a lot of Node.js tools for doing this.

He then demonstrated a cloud function that allowed the flattening of the Firebase Database: every time a write came in, an index was automatically generated in another place in the database.

He also showed off their website which was built with Firebase Hosting and discussed some finer points about performance.

All in all the talk was very thick with technical details and with the nitty-gritty details of running a production app based by Firebase Database technologies. I get the impression that it is indeed possible, although it does require a lot of effort to keep it running.

Conclusion
----------

It was a nice evening talk and it's nice to see where Google intends to take Firebase in the future. Firebase is here to stay and some of its components are sure to become mainstream soon enough.