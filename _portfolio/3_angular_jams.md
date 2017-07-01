---
layout: post
title: Bloc Jams
thumbnail-path: "img/bj-1.png"
short-description: An online digital streaming audio player inspired by Spotify.

---

{:.center}
![]({{ site.baseurl }}/img/bj-1.png)

![](https://github.com/favicon.ico) &nbsp;&nbsp;[View on GitHub](https://github.com/davelively14/bloc-jams-angular/)

## Explanation

As part of the Foundation Frontend Web Development portion of Bloc's [Software Developer Track](https://www.bloc.io/software-developer-track), we were tasked to develop an application that works similar to Spotify.

## Problem

Users must be able to do the following:
- Play songs, to include adjusting volume, seeking, and autoplay next song
- Rank albums and favorite songs
- Login and maintain user information
- Search for songs, albums, and artists

## Solution

For my technology stack, we used [AngularJS](https://angularjs.org/) to develop the single page app with [Grunt](http://gruntjs.com/) as the JavaScript task runner. No persistence of the data was necessary, so it all lives and dies with the session. For controlling the songs, we used [Buzz!](http://buzz.jaysalvat.com/), a small JavaScript HTML5 audio library.

#### A note on route configuration

We opted to use [AngularUI Router](https://github.com/angular-ui/ui-router) in lieu of the built in AngularJS routing system due to it's flexibility and additional features. With UI-Router, an application can be in different states that determine what to display when a user navigates to a specific route. UI-Router will take care of replacing the contents of <ui-view></ui-view> with a template when a user navigates to the proper route. Each template can be unique, while the shared code is kept in the global file.


#### Play songs

![](/img/bj-2.png)

In order to play songs from an album effectively and without interruption, we created a player bar that is included on the `user`, `collection`, `album`, and `search` templates. The songs can be selected and played from the `album`. We serve the data for the album via the `Fixtures.js` service and manage the state of the song player via `SongPlayer.js` service.

We manage the seek and volume bars via the `seek-bar.js` directive. We can customize the seek bars via passing attributes within the HTML:

{% highlight html %}

<seek-bar
  value="{ { playerBar.songPlayer.currentTime } }"
  max="{ { playerBar.songPlayer.currentSong.duration} }"
  on-change="playerBar.songPlayer.setCurrentTime(value)">
</seek-bar>

...

<seek-bar
  value="{ { playerBar.songPlayer.volume } }"
  max="{ { 100 } }"
  on-change="playerBar.songPlayer.setVolume(value)">
</seek-bar>
{% endhighlight %}

We utilize the Buzz library function to autoplay the next song:

{% highlight javascript %}

currentBuzzObject.bind('ended', function() {
  SongPlayer.next();
});

{% endhighlight %}

#### Rank albums and favorite songs

![](/img/bj-3.png)

To rate the albums, we added rank key value pairs to the album object in the `Fixtures.js` service and then dynamically generate the rating of the album via `ng-repeat` and `ng-show`:

{% highlight html %}

<span ng-repeat="n in [1, 2, 3, 4, 5]" class="rating">
  <a
    ng-click="setRating($index + 1)"
    ng-show="albumData.rating > $index">
    <span class="ion-ios-star"></span>
  </a>
  <a
    ng-click="setRating($index + 1)"
    ng-show="albumData.rating <= $index">
    <span class="ion-ios-star-outline"></span>
  </a>
</span>

{% endhighlight %}

For each song, we added a similar rank key value pair, but we only generated one star:

{% highlight html %}

<td class="rating">
  <a ng-click="setSongRating($index, 0)" ng-show="song.rating > 0"><span class="ion-ios-star"></span></a>
  <a ng-click="setSongRating($index, 1)" ng-show="song.rating < 1"><span class="ion-ios-star-outline"></span></a>
</td>

{% endhighlight %}

#### Login and maintain user information

<img align="right" src="/img/bj-4.png" width="600">
Via the `user.html` template, we allow the user to create a username, provide an email, and set their favorite band. We store that information in an object maintained in the `User.js` service.

Per the requirements, we were not required to hansh and store any password for this application.

#### Search for songs, albums, and artists

Instead of searching for album, song, or artist individually, we opted to provide the user the opportunity to provide one search string and return the results for _any_ result that matched the string:

{% highlight javascript %}

function SearchCtrl($scope, Fixtures) {
  var albums = Fixtures.getAllAlbums();
  $scope.albumResults = [];
  $scope.songResults = [];
  $scope.artistResults = [];

  $scope.submit = function() {
    $scope.albumResults = [];
    $scope.songResults = [];
    $scope.artistResults = [];

    var str = $scope.text;

    for (var i = 0; i < albums.length; i++) {
      if (albums[i].title.toLowerCase().search(str.toLowerCase()) >= 0) {
        $scope.albumResults.push(albums[i]);
      }

      if (albums[i].artist.toLowerCase().search(str.toLowerCase()) >= 0) {
        $scope.artistResults.push(albums[i]);
      }

      for (var song = 0; song < albums[i].songs.length; song++) {
        if (albums[i].songs[song].title.toLowerCase().search(str.toLowerCase()) >= 0) {
          $scope.songResults.push({song: albums[i].songs[song], album: albums[i]});
        }
      }
    }
  }
}

{% endhighlight %}

## Results

Users are able to login, find their songs of choice, and jam out.

## Conclusion

While there may be more powerful online music stores, AngularJS, combined with the AngularUI router, provides a powerful framework to create dynamic, Single Page Applications.
