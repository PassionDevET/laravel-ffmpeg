# Laravel FFMpeg

[![Latest Version on Packagist](https://img.shields.io/packagist/v/pbmedia/laravel-ffmpeg.svg?style=flat-square)](https://packagist.org/packages/pbmedia/laravel-ffmpeg)
[![Software License](https://img.shields.io/badge/license-MIT-brightgreen.svg?style=flat-square)](LICENSE.md)
[![Build Status](https://img.shields.io/travis/pascalbaljetmedia/laravel-ffmpeg/master.svg?style=flat-square)](https://travis-ci.org/pascalbaljetmedia/laravel-ffmpeg)
[![Quality Score](https://img.shields.io/scrutinizer/g/pascalbaljetmedia/laravel-ffmpeg.svg?style=flat-square)](https://scrutinizer-ci.com/g/pascalbaljetmedia/laravel-ffmpeg)
[![Total Downloads](https://img.shields.io/packagist/dt/pbmedia/laravel-ffmpeg.svg?style=flat-square)](https://packagist.org/packages/pbmedia/laravel-ffmpeg)

This package provides an integration with FFmpeg for Laravel 6.0 and higher. [Laravel's Filesystem](http://laravel.com/docs/7.0/filesystem) handles the storage of the files.

## Features
* Super easy wrapper around [PHP-FFMpeg](https://github.com/PHP-FFMpeg/PHP-FFMpeg), including support for filters and other advanced features.
* Integration with [Laravel's Filesystem](http://laravel.com/docs/7.0/filesystem), [configuration system](https://laravel.com/docs/7.0/configuration) and [logging handling](https://laravel.com/docs/7.0/errors).
* Compatible with Laravel 6.0 and higher, support for [Package Discovery](https://laravel.com/docs/7.0/packages#package-discovery).
* Built-in support for HLS.
* Built-in support for concatenation, multiple inputs/outputs, image sequences (timelapse), complex filters (and mapping), frame/thumbnail exports.
* PHP 7.2 and higher.

## Video Course Builder
We're developing a package/app/service to set up your course platform quickly. We have one straightforward question, however. Which solution and pricing would you prefer? Vote at [videocoursebuilder.com](https://videocoursebuilder.com) and read all about the features!

Are there other features you want to see? Do you have ideas about pricing models? Or do you have other suggestions? Let me know [right here on GitHub](https://github.com/pascalbaljetmedia/laravel-ffmpeg/issues/187) or reach out [on Twitter](https://twitter.com/pascalbaljet).

## Installation

You can install the package via composer:

```bash
composer require pbmedia/laravel-ffmpeg
```

Add the Service Provider and Facade to your ```app.php``` config file if you're not using Package Discovery.

```php
// config/app.php

'providers' => [
    ...
    ProtoneMedia\LaravelFFMpeg\Support\ServiceProvider::class,
    ...
];

'aliases' => [
    ...
    'FFMpeg' => ProtoneMedia\LaravelFFMpeg\Support\FFMpeg::class
    ...
];
```

Publish the config file using the artisan CLI tool:

```bash
php artisan vendor:publish --provider="ProtoneMedia\LaravelFFMpeg\Support\ServiceProvider"
```

## Upgrading to v7

* The namespace has changed to `ProtoneMedia\LaravelFFMpeg`, the facade has been renamed to `ProtoneMedia\LaravelFFMpeg\Support\FFMpeg`, and the Service Provider has been renamed to `ProtoneMedia\LaravelFFMpeg\Support\ServiceProvider`.
* Chaining exports are still supported, but you have to reapply filters for each export.
* HLS playlists now include bitrate, framerate and resolution data. The segments also use a new naming pattern ([read more](#using-custom-segment-patterns)). Please verify your exports still work in your player.
* HLS export is now executed as *one* job instead of exporting each format/stream separately. This uses FFMpeg's `map` and `filter_complex` features. It might be sufficient to replace all calls to `addFilter` with `addLegacyFilter`, but some filters should be migrated manually. Please read the [documentation on HLS](#hls) to find out more about adding filters.

## Usage

Convert an audio or video file:

```php
FFMpeg::fromDisk('songs')
    ->open('yesterday.mp3')
    ->export()
    ->toDisk('converted_songs')
    ->inFormat(new \FFMpeg\Format\Audio\Aac)
    ->save('yesterday.aac');
```

Instead of the ```fromDisk()``` method you can also use the ```fromFilesystem()``` method, where ```$filesystem``` is an instance of ```Illuminate\Contracts\Filesystem\Filesystem```.

```php
$media = FFMpeg::fromFilesystem($filesystem)->open('yesterday.mp3');
```

### Progress monitoring

You can monitor the transcoding progress. Use the ```onProgress``` method to provide a callback, which gives you the completed percentage. In previous versions of this package you had to pass the callback to the format object.

```php
FFMpeg::open('steve_howe.mp4')
    ->export()
    ->onProgress(function ($percentage) {
        echo "{$percentage}% transcoded";
    });
```

As of version 7.0, the callback also exposes `$remaining` (in seconds) and `$rate`:

```php
FFMpeg::open('steve_howe.mp4')
    ->export()
    ->onProgress(function ($percentage, $remaining, $rate) {
        echo "{$remaining} seconds left at rate: {$rate}";
    });
```

### Filters

You can add filters through a ```Closure``` or by using PHP-FFMpeg's Filter objects:

```php
use FFMpeg\Filters\Video\VideoFilters;

FFMpeg::fromDisk('videos')
    ->open('steve_howe.mp4')
    ->addFilter(function (VideoFilters $filters) {
        $filters->resize(new \FFMpeg\Coordinate\Dimension(640, 480));
    })
    ->export()
    ->toDisk('converted_videos')
    ->inFormat(new \FFMpeg\Format\Video\X264)
    ->save('small_steve.mkv');

// or

$start = \FFMpeg\Coordinate\TimeCode::fromSeconds(5)
$clipFilter = new \FFMpeg\Filters\Video\ClipFilter($start);

FFMpeg::fromDisk('videos')
    ->open('steve_howe.mp4')
    ->addFilter($clipFilter)
    ->export()
    ->toDisk('converted_videos')
    ->inFormat(new \FFMpeg\Format\Video\X264)
    ->save('short_steve.mkv');
```

As of version 7.0, you can also call the `addFilter` method *after* the `export` method:

```php
use FFMpeg\Filters\Video\VideoFilters;

FFMpeg::fromDisk('videos')
    ->open('steve_howe.mp4')
    ->export()
    ->toDisk('converted_videos')
    ->inFormat(new \FFMpeg\Format\Video\X264)
    ->addFilter(function (VideoFilters $filters) {
        $filters->resize(new \FFMpeg\Coordinate\Dimension(640, 480));
    })
    ->save('small_steve.mkv');
```

### Custom filters

Sometimes you don't want to use the built-in filters. You can apply your own filter by providing a set of options. This can be an array or multiple strings as arguments:

```php
FFMpeg::fromDisk('videos')
    ->open('steve_howe.mp4')
    ->addFilter(['-itsoffset', 1]);

// or

FFMpeg::fromDisk('videos')
    ->open('steve_howe.mp4')
    ->addFilter('-itsoffset', 1);
```

### Chain multiple convertions

```php
// The 'fromDisk()' method is not required, the file will now
// be opened from the default 'disk', as specified in
// the config file.

FFMpeg::open('my_movie.mov')

    // export to FTP, converted in WMV
    ->export()
    ->toDisk('ftp')
    ->inFormat(new \FFMpeg\Format\Video\WMV)
    ->save('my_movie.wmv')

    // export to Amazon S3, converted in X264
    ->export()
    ->toDisk('s3')
    ->inFormat(new \FFMpeg\Format\Video\X264)
    ->save('my_movie.mkv');

    // you could even discard the 'toDisk()' method,
    // now the converted file will be saved to
    // the same disk as the source!
    ->export()
    ->inFormat(new FFMpeg\Format\Video\WebM)
    ->save('my_movie.webm')

    // optionally you could set the visibility
    // of the exported file
    ->export()
    ->inFormat(new FFMpeg\Format\Video\WebM)
    ->withVisibility('public')
    ->save('my_movie.webm')
```

### Create a frame from a video

```php
FFMpeg::fromDisk('videos')
    ->open('steve_howe.mp4')
    ->getFrameFromSeconds(10)
    ->export()
    ->toDisk('thumnails')
    ->save('FrameAt10sec.png');

// Instead of the 'getFrameFromSeconds()' method, you could
// also use the 'getFrameFromString()' or the
// 'getFrameFromTimecode()' methods:

$media = FFMpeg::open('steve_howe.mp4');
$frame = $media->getFrameFromString('00:00:13.37');

// or

$timecode = new FFMpeg\Coordinate\TimeCode(...);
$frame = $media->getFrameFromTimecode($timecode);
```

You can also get the raw contents of the frame instead of saving it to the filesystem:

```php
$contents = FFMpeg::open('video.mp4')
    ->getFrameFromSeconds(2)
    ->export()
    ->getFrameContents();
```

### Create a timelapse

You can create a timelapse from a sequence of images by using the `asTimelapseWithFramerate` method on the exporter

```php
FFMpeg::open('feature_%04d.png')
    ->export()
    ->asTimelapseWithFramerate(1)
    ->inFormat(new X264)
    ->save('timelapse.mp4');
```

### Multiple inputs

As of version 7.0 you can open multiple inputs, even from different disks. This uses FFMpeg's `map` and `filter_complex` features. You can open multiple files by chaining the `open` method of by using an array. You can mix inputs from different disks.

```php
FFMpeg::open('video1.mp4')->open('video2.mp4');

FFMpeg::open(['video1.mp4', 'video2.mp4']);

FFMpeg::fromDisk('uploads')
    ->open('video1.mp4')
    ->fromDisk('archive')
    ->open('video2.mp4');
```

When you open multiple inputs, you have to add mappings so FFMpeg knows how to route them. This package provides a `addFormatOutputMapping` method, which takes three parameters: the format, the output, and the output labels of the `-filter_complex` part.

The output (2nd argument) should be an instanceof `\ProtoneMedia\LaravelFFMpeg\Filesystem\Media`. You can instantiate with the `make` method, call it with the name of the disk and the path (see example).

Check out this example, which maps separate video and audio inputs into one output.

```php
FFMpeg::fromDisk('local')
    ->open(['video.mp4', 'audio.m4a'])
    ->export()
    ->addFormatOutputMapping(new X264, Media::make('local', 'new_video.mp4'), ['0:v', '1:a'])
    ->save();
```

This is an example [from the underlying library](https://github.com/PHP-FFMpeg/PHP-FFMpeg#base-usage):

```php
// This code takes 2 input videos, stacks they horizontally in 1 output video and
// adds to this new video the audio from the first video. (It is impossible
// with a simple filter graph that has only 1 input and only 1 output).

FFMpeg::fromDisk('local')
    ->open(['video.mp4', 'video2.mp4'])
    ->export()
    ->addFilter('[0:v][1:v]', 'hstack', '[v]')  // $in, $parameters, $out
    ->addFormatOutputMapping(new X264, Media::make('local', 'stacked_video.mp4'), ['0:a', '[v]'])
    ->save();
```

Just like single inputs, you can also pass a callback to the `addFilter` method. This will give you an instance of `\FFMpeg\Filters\AdvancedMedia\ComplexFilters`:

```php
use FFMpeg\Filters\AdvancedMedia\ComplexFilters;

FFMpeg::open(['video.mp4', 'video2.mp4'])
    ->export()
    ->addFilter(function(ComplexFilters $filters) {
        // $filters->watermark(...);
    });
```

### Concat files without transcoding

```php
FFMpeg::fromDisk('local')
    ->open(['video.mp4', 'video2.mp4'])
    ->export()
    ->concatWithoutTranscoding()
    ->save('concat.mp4');
```

### Concat files with transcoding

```php
FFMpeg::fromDisk('local')
    ->open(['video.mp4', 'video2.mp4'])
    ->export()
    ->inFormat(new X264)
    ->concatWithTranscoding($hasVideo = true, $hasAudio = true)
    ->save('concat.mp4');
```

### Determinate duration

With the ```Media``` class you can determinate the duration of a file:

```php
$media = FFMpeg::open('wwdc_2006.mp4');

$durationInSeconds = $media->getDurationInSeconds(); // returns an int
$durationInMiliseconds = $media->getDurationInMiliseconds(); // returns a float
```

### Handling remote disks

When opening or saving files from or to a remote disk, temporary files will be created on your server. After you're done exporting or processing these files, you could clean them up by calling the ```cleanupTemporaryFiles()``` method:

```php
FFMpeg::cleanupTemporaryFiles();
```

## HLS

You can create a M3U8 playlist to do [HLS](https://en.wikipedia.org/wiki/HTTP_Live_Streaming).

```php
$lowBitrate = (new X264)->setKiloBitrate(250);
$midBitrate = (new X264)->setKiloBitrate(500);
$highBitrate = (new X264)->setKiloBitrate(1000);

FFMpeg::fromDisk('videos')
    ->open('steve_howe.mp4')
    ->exportForHLS()
    ->setSegmentLength(10) // optional
    ->setKeyFrameInterval(48) // optional
    ->addFormat($lowBitrate)
    ->addFormat($midBitrate)
    ->addFormat($highBitrate)
    ->save('adaptive_steve.m3u8');
```

The ```addFormat``` method of the HLS exporter takes an optional second parameter which can be a callback method. This allows you to add different filters per format. First, check out the *Multiple inputs* section to understand how complex filters are handled.

You can use the `addFilter` method to add a complex filter (see `$lowBitrate` example). Since the `scale` filter is used a lot, there is a helper method (see `$midBitrate` example). You can also use a callable to get access to the `ComplexFilters` instance. The package provides the `$in` and `$out` arguments so you don't have to worry about it (see `$highBitrate` example).

As of version 7.0, HLS export is built using FFMpeg's `map` and `filter_complex` features. This is a breaking change from previous versions which performed a single export for each format. If you're upgrading, replace the `addFilter` calls with `addLegacyFilter` calls and verify the result (see `$superBitrate` example). Not all filters will work this way and some need to be upgraded manually.

```php
$lowBitrate = (new X264)->setKiloBitrate(250);
$midBitrate = (new X264)->setKiloBitrate(500);
$highBitrate = (new X264)->setKiloBitrate(1000);
$superBitrate = (new X264)->setKiloBitrate(1500);

FFMpeg::open('steve_howe.mp4')
    ->exportForHLS()
    ->addFormat($lowBitrate, function($media) {
        $media->addFilter('scale=640:480');
    })
    ->addFormat($midBitrate, function($media) {
        $media->scale(960, 720);
    })
    ->addFormat($highBitrate, function ($media) {
        $media->addFilter(function ($filters, $in, $out) {
            $filters->custom($in, 'scale=1920:1200', $out); // $in, $parameters, $out
        });
    })
    ->addFormat($superBitrate, function($media) {
        $media->addLegacyFilter(function ($filters) {
            $filters->resize(new \FFMpeg\Coordinate\Dimension(2560, 1920));
        });
    })
    ->save('adaptive_steve.m3u8');
```

### Using custom segment patterns

You can use a custom pattern to name the segments and playlists. The `useSegmentFilenameGenerator` gives you 5 arguments. The first, second and third argument provide information about the basename of the export, the format of the current iteration and the key of the current iteration. The fourth argument is a callback you should call with your *segments* pattern. The fifth argument is a callback you should call with your *playlist* pattern. Note that this is not the name of the master playlist, but the name of the playlist of each format.

```php
FFMpeg::fromDisk('videos')
    ->open('steve_howe.mp4')
    ->exportForHLS()
    ->useSegmentFilenameGenerator(function ($name, $format, $key, callable $segments, callable $playlist) {
        $segments("{$name}-{$format}-{$key}-%03d.ts");
        $playlist("{$name}-{$format}-{$key}.m3u8");
    });
```

## Advanced

The Media object you get when you 'open' a file, actually holds the Media object that belongs to the [underlying driver](https://github.com/PHP-FFMpeg/PHP-FFMpeg). It handles dynamic method calls as you can see [here](https://github.com/pascalbaljetmedia/laravel-ffmpeg/blob/master/src/Media.php#L114-L117). This way all methods of the underlying driver are still available to you.

```php
// This gives you an instance of ProtoneMedia\LaravelFFMpeg\MediaOpener
$media = FFMpeg::fromDisk('videos')->open('video.mp4');

// The 'getStreams' method will be called on the underlying Media object since
// it doesn't exists on this object.
$codec = $media->getStreams()->first()->get('codec_name');
```

If you want direct access to the underlying object, call the object as a function (invoke):

```php
// This gives you an instance of ProtoneMedia\LaravelFFMpeg\MediaOpener
$media = FFMpeg::fromDisk('videos')->open('video.mp4');

// This gives you an instance of FFMpeg\Media\MediaTypeInterface
$baseMedia = $media();
```

## Experimental

The [progress listener](#progress-monitoring) exposes the transcoded percentage, but the underlying package also has an internal `AbstractProgressListener` that exposes the current pass and the current time. Though the use-case is limited, you might want to get access to this listener instance. You can do this by decorating the format with the `ProgressListenerDecorator`. This feature is highly experimental, so be sure the test this thoroughly before using it in production.

```php
use FFMpeg\Format\ProgressListener\AbstractProgressListener;
use ProtoneMedia\LaravelFFMpeg\FFMpeg\ProgressListenerDecorator;

$format = new \FFMpeg\Format\Video\X264;
$decoratedFormat = ProgressListenerDecorator::decorate($format);

FFMpeg::open('video.mp4')
    ->export()
    ->inFormat($decoratedFormat)
    ->onProgress(function () use ($decoratedFormat) {
        $listeners = $decoratedFormat->getListeners();  // array of listeners

        $listener = $listeners[0];  // instance of AbstractProgressListener

        $listener->getCurrentPass();
        $listener->getTotalPass();
        $listener->getCurrentTime();
    })
    ->save('new_video.mp4');
```

## Example app

Here's a blog post that will help you get started with this package:

https://protone.media/en/blog/how-to-use-ffmpeg-in-your-laravel-projects

## Wiki

* [Custom filters](https://github.com/pascalbaljetmedia/laravel-ffmpeg/wiki/Custom-filters)
* [FFmpeg failed to execute command](https://github.com/pascalbaljetmedia/laravel-ffmpeg/wiki/FFmpeg-failed-to-execute-command)
* [Get the dimensions of a Video file](https://github.com/pascalbaljetmedia/laravel-ffmpeg/wiki/Get-the-dimensions-of-a-Video-file)
* [Monitoring the transcoding progress](https://github.com/pascalbaljetmedia/laravel-ffmpeg/wiki/Monitoring-the-transcoding-progress)
* [Unable to load FFProbe](https://github.com/pascalbaljetmedia/laravel-ffmpeg/wiki/Unable-to-load-FFProbe)

## Changelog

Please see [CHANGELOG](CHANGELOG.md) for more information about what has changed recently.

## Testing

```bash
$ composer test
```

## Contributing

Please see [CONTRIBUTING](CONTRIBUTING.md) for details.

## Other Laravel packages

* [`Laravel Analytics Event Tracking`](https://github.com/pascalbaljetmedia/laravel-analytics-event-tracking): Laravel package to easily send events to Google Analytics.
* [`Laravel Blade On Demand`](https://github.com/pascalbaljetmedia/laravel-blade-on-demand): Laravel package to compile Blade templates in memory.
* [`Laravel Paddle`](https://github.com/pascalbaljetmedia/laravel-paddle): Paddle.com API integration for Laravel with support for webhooks/events.
* [`Laravel Verify New Email`](https://github.com/pascalbaljetmedia/laravel-verify-new-email): This package adds support for verifying new email addresses: when a user updates its email address, it won't replace the old one until the new one is verified.
* [`Laravel WebDAV`](https://github.com/pascalbaljetmedia/laravel-webdav): WebDAV driver for Laravel's Filesystem.
## Security

If you discover any security-related issues, please email code@protone.media instead of using the issue tracker. Please do not email any questions, open an issue if you have a question.

## Credits

- [Pascal Baljet](https://github.com/pascalbaljet)
- [All Contributors](../../contributors)

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.
