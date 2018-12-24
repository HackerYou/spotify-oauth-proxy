# Spotify oAuth Proxy

[![Deploy](https://www.herokucdn.com/deploy/button.svg)](https://heroku.com/deploy)

This is a deployable server that allows you to use either the Spotify Authorization Code Flow or Client Credentials Flow, or both, to create an application with the need to build your own server.

Key | Value
---  | ---
CLIENT_SECRET | Your client secret
CLIENT_ID | Your client id
APP_URL | URL for your front-end app
SERVER_URL | URL for the deployed herokuapp
SCOPES | The authorization scopes you want the user to grant; scopes should be separated with a space. A full list of available scopes can be found [here](https://developer.spotify.com/documentation/general/guides/scopes/). If no scopes are specified, authorization will be granted only to access publicly available information.

When you deploy to Heroku go to your application in the dashboard and click on the settings tab. There you can click `Reveal Config Vars`, and make sure you add the five from above.

## Workflow

Here is the workflow for working with this server in your front-end application.

Create a link that goes to the deployed servers URL and the `/auth` endpoint.

```html
<a href="https://someapp.herokuapp.com/auth">Login to Spotify</a>
```

Where `https://someapp.herokuapp.com` is your URL for the heroku server.

This will send the request to Spotify, which then redirects to the servers `/redirect` endpoint. From there it will take the returned data and then send the proper info to the next stop, which is the token step. Once that is completed it will redirect to your application with an object that looks like this as a query string.

On Spotify in the application dashboard for the API click the `edit settings` button and make sure you add `https://someapp.herokuapp.com/redirect` in the redirects URI's section.

```js
{
    access_token: "BQC5M6sBvrZBV3PFcICDKYyTnlfbbQ6...8WLXdEok",
    token_type: "Bearer",
    expires_in: 3600,
    refresh_token: "AQA1hxMB8FwWz3oqTegdK809jrYuVp-K...4AFgJoo"
}
```

However in order to get this, we will need to take the query string and grab what we need from it. In your JS on load you should have something like.

```js
const URL = new URL(location.href);
const tokens = {
    access_token: url.searchParams.get('access_token'),
    refresh_token: url.searchParams.get('refresh_token')
};
```

Here is a one page example of using the proxy, assume that this is running on `localhost:3400` and that has been added as the `APP_URL` for the proxy on heroku.

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Spotify Proxy Test</title>
</head>
<body>
    <!-- The anchor tag to kick off the log in -->
    <a href="https://spotify-proxy-test.herokuapp.com/auth">Login to Spotify</a>
    <button>Get info about me!</button>
    <script src='https://code.jquery.com/jquery-3.2.1.min.js' integrity='sha256-hwg4gsxgFZhOsEEamdOYGBf13FyQuiTwlAQgxVSNgt4='crossorigin='anonymous'></script>

    <script>
        const app = {};
        //Object to store our tokens
        app.tokenInfo = {};
        //This method is used to get the token for every request. Since a token only lasts for 3600ms we need to get a new token for each request
        app.getToken = () => {
            //Return a promise
            return new Promise((resolve,reject) => {
                //Make the request to get the token
                $.ajax({
                    url: 'https://spotify-proxy-test.herokuapp.com/refresh',
                    data: {
                        //Use the refresh_token we got on load.
                        refresh_token: app.tokenInfo.refresh_token
                    }
                })
                .then((res) => {
                    //Grab the new token and return it.
                    const { access_token } = JSON.parse(res);
                    resolve(access_token);
                }); 
            });
        };

        app.getMe = () => {
            //Before each call, use the getToken method to get a new token
            app.getToken()
                .then((token) => {
                    //then use the token in a header
                    $.ajax({
                        url: 'https://api.spotify.com/v1/me',
                        headers: {
                            'Authorization' : `Bearer ${token}`
                        }
                    })
                    .then(console.log);
                });
        };

        app.events = () => {
            $('button').on('click',() => {
                app.getMe();
            });
        };     

        app.init = () => {
            //On load of the app check to see if there is a query string in the URL.
            //If not, the user needs to click the a tag in the page
            if(location.search !== "") {
                const url = new URL(location.href);
                //Grab the info
                app.tokenInfo = {
                    access_token: url.searchParams.get('access_token'),
                    refresh_token: url.searchParams.get('refresh_token')
                }
                //call any events
                app.events();
            }
        }

        $(app.init);
    </script>
</body>
</html>
```