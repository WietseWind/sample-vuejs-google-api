# Google API with VueJS


Google API (Drive, Calendar) with window.fetch and VueJS (Sample)

More on Vue:
[https://xrpcommunity.blog/develop-awesome-webapps-using-vuejs-webpack-demonstrating-by-building-a-xrp-ledger-integration/](https://xrpcommunity.blog/develop-awesome-webapps-using-vuejs-webpack-demonstrating-by-building-a-xrp-ledger-integration/)

# Setup

- Get a Webapp client id from the [Google API Console](https://console.cloud.google.com/apis/credentials)
- Make sure you have the [**Drive**](https://console.cloud.google.com/apis/library/drive.googleapis.com/?q=drive) and [**Calendar**](https://console.cloud.google.com/apis/library/calendar-json.googleapis.com/?q=calendar) libs enabled in the Google API Console
- Put it at [line 34 in `src/App.vue`](https://github.com/WietseWind/sample-vuejs-google-api/blob/master/src/App.vue#L34)


## Build Setup

``` bash
# install dependencies
npm install

# serve with hot reload at localhost:8080
npm run dev

# build for production with minification
npm run build

# build for production and view the bundle analyzer report
npm run build --report
```
