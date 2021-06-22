---
layout: post
title: Reed 基于miniflux的RSS应用
date: 2020-08-15 11:13:30
tags: Flutter
---

Simple RSS Reader based on [Miniflux](https://miniflux.app/) API

<!-- more -->

## Prerequirement

- follow [Miniflux Documentation](https://miniflux.app/docs/installation.html) to deploy your site
- get api key from `Settings -> API Keys` of your site

## Download

go to [release page](https://github.com/ali322/reed/releases) download app

## ScreenShot

- Lignt Mode

![1](https://raw.githubusercontent.com/ali322/reed/master/screenshot/1.jpg)
![2](https://raw.githubusercontent.com/ali322/reed/master/screenshot/2.jpg)

- Dark Mode

![3](https://raw.githubusercontent.com/ali322/reed/master/screenshot/3.jpg)
![4](https://raw.githubusercontent.com/ali322/reed/master/screenshot/4.jpg)

## Features

- support dark mode
- i18n enabled
- simple and clean theme

## Develop
make sure finish [install Flutter](https://flutter.io/get-started/install/) successful

1. clone this repo
`git clone https://github.com/ali322/reed`
2. install all the packages
`flutter packages get`
3. run the app in simulator on your own
`flutter run`

## Packages in using
Reed build on following packages
* [http](https://pub.dev/packages/http)
* [bloc](https://pub.dev/packages/bloc)
* [flutter_html](https://pub.dev/packages/flutter_html)
* [easy_localization](https://pub.dev/packages/easy_localization)
* [flutter_secure_storage](https://pub.dev/packages/flutter_secure_storage)
* [url_launcher](https://pub.dev/packages/url_launcher)


## Todo

- finish more scenes
- fix some unknow bugs


## License

[MIT License](http://en.wikipedia.org/wiki/MIT_License)