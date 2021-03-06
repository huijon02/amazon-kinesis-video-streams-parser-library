# Amazon Kinesis Video Streams Parser Library 

## License

This library is licensed under the Apache 2.0 License. 

## Introduction
The Amazon Kinesis Video Streams Parser Library for Java enables Java developers to parse the streams returned by `GetMedia` calls to Amazon Kinesis Video. 
It contains:
* A streaming Mkv Parser called `StreamingMkvReader` that provides an iterative interface to read the `MkvElement`s in a stream.
* Applications such as `OutputSegmentMerger` and `FragmentMetadataVisitor` built using the `StreamingMkvReader` .
* A callback based parser called `EBMLParser` that minimizes data buffering and copying. `StreamingMkvReader` is built on top of `EBMLParser`
* Unit tests for the applications and parsers that demonstrate how the applications work.

## Building from Source
After you've downloaded the code from GitHub, you can build it using Maven. Use this command: `mvn clean install`


## Details
### StreamingMkvReader
`StreamingMkvReader` which provides an iterative interface to read `MkvElement`s from a stream.
A caller calls `nextIfAvailable` to get the next `MkvElement`. An `MkvElement` wrapped in an `Optional` is returned if a complete element is available.
It buffers an individual `MkvElement` until it can return a complete `MkvElement`.  
The `mightHaveNext` method returns true if there is a chance that additional `MkvElements` can be returned. 
It returns false when the end of the input stream has been reached.
 
### MkvElement
There are three types of `MkvElement` vended by a `MkvStreamReader`:
* `MkvDataElement`: This encapsulates Mkv Elements that are not master elements and contain data. 
* `MkvStartMasterElement` : This represents the start of a *master* Mkv element that contains child elements. Child elements can be other master elements or data elements.
* `MkvEndMasterElement` : This represents the end of *master* element that contains child elements.

### MkvElementVisitor
The `MkvElementVisitor` is a visitor pattern that helps process the events in the `MkvElement` hierarchy. It has a visit 
method for each type of `MkvElement`.

## Visitors
A `GetMedia` call to Kinesis Video vends a stream of fragments where each fragment is encapsulated in a Mkv stream containing *EBML* and *Segment* elements.
`OutputSegmentMerger` can be used to merge consecutive fragments that share the same *EBML* and *Track* data into a single Mkv stream with 
a shared *EBML* and *Segment*. This is useful for passing the output of `GetMedia` to any downstream processor that expects a single Mkv stream
with one *Segment*. Its use can be seen in `OutputSegmentMergerTest`

`FragmentMetadataVisitor` is a `MkvElementVisitor` that collects the Kinesis Video specific meta-data (such as *FragmentNumber* and *Server Side Timestamp* )
 for the current fragment being processed. The `getCurrentFragmentMetadata` method can be used to get the current fragment's metadata. Similarly 
`getPreviousFragmentMetadata` can be used get the previous fragment's metadata. The `getMkvTrackMetadata` method can be used to get
the details of a particular track.

`ElementSizeAndOffsetVisitor` is a visitor that writes out the metadata of the Mkv elements in a stream. For each element
 the name, offset, header size and data size is written out. The output uses indentation to indicate the hierarchy of master elements
 and their child elements. `ElementSizeAndOffsetVisitor` is useful for looking into Mkv streams, where mkvinfo fails.

`CountVisitor` is a visitor that can be used to count the number of Mkv elements of different types in a Mkv stream.

`CompositeMkvElementVisitor` is a visitor that is made up of a number of constituent visitors. It calls accept on the 
visited `MkvElement` for each constituent visitor in the order in which the visitors are specified.

`FrameVisitor` is a visitor used to process the frames in the output of a GetMedia call. It invokes an implementation of the
 `FrameVisitor.FrameProcessor` and provides it with a `Frame` object and the metadata of the track to which the `Frame` belongs.

`CopyVisitor` is a visitor used to copy the raw bytes of the Mkv elements in a stream to an output stream.

## ResponseStreamConsumers
The `GetMediaResponseStreamConsumer` is an abstract class used to consume the output of a GetMedia* call to Kinesis Video in a streaming fashion.
It supports a single abstract method called process that is invoked to process the streaming payload of a GetMedia response.
The first parameter for process method is the payload inputStream in a GetMediaResult returned by a call to GetMedia.
Implementations of the process method of this interface should block until all the data in the inputStream has been
 processed or the process method decides to stop for some other reason. The second argument is a FragmentMetadataCallback 
 which is invoked at the end of every processed fragment. The `GetMediaResponseStreamConsumer` provides a utility method 
 `processWithFragmentEndCallbacks` that can be used by child classes to  implement the end of fragment callbacks.
 The process method can be implemented using a combination of the visitors described earlier.
 
### MergedOutputPiper
The `MergedOutputPiper` extends `GetMediaResponseStreamConsumer` to merge consecutive mkv streams in the output of GetMedia
 and pipes the merged stream to the stdin of a child process. It is meant to be used to pipe the output of a GetMedia* call to a processing application that can not deal
with having multiple consecutive mkv streams. Gstreamer is one such application that requires a merged stream.


## Example
* `KinesisVideoExample` is an example that shows how the `StreamingMkvReader` and the different visitors can be integrated 
with the AWS SDK for the Kinesis Video. This example provides examples for

    * Create a stream, deleting and recreating if the stream of the same name already exists.
    * Call PutMedia to stream video fragments into the stream.
    * Simultaneously call GetMedia to stream video fragments out of the stream.
    * It uses the StreamingMkvParser to parse the returned the stream and apply the `OutputSegmentMerger`, `FragmentMetadataVisitor` visitors
 along with a local one as part of the same `CompositeMkvElementVisitor` visitor.
 
* `KinesisVideoRendererExample` shows parsing and rendering of KVS video stream fragments using JCodec(http://jcodec.org/) that were ingested using Producer SDK GStreamer sample application.
    * To run the example:    
      Run the Unit test `testExample` in `KinesisVideoRendererExampleTest`. After starting the unitTest you should be able to view the frames in a JFrame.
    * If you want to store it as image files you could do it by adding (in KinesisVideoRendererExample after AWTUtil.toBufferedImage(rgb, renderImage); )
 
    ```
    try {
        ImageIO.write(renderImage, "png", new File(String.format("frame-capture-%s.png", UUID.randomUUID())));
     } catch (IOException e) {
        log.warn("Couldn't convert to a PNG", e);
    }
    ``` 
    * It has been tested not only for streams ingested by `PutMediaWorker` but also streams sent to Kinesis Video Streams using GStreamer Demo application (https://github.com/awslabs/amazon-kinesis-video-streams-producer-sdk-cpp)    

* `KinesisVideoGStreamerPiperExample` is an example for continuously piping the output of GetMedia calls from a Kinesis Video stream to GStreamer.
 The test `KinesisVideoGStreamerPiperExampleTest` provides an example that pipes the output of a KVS GetMedia call to a Gstreamer pipeline.
 The Gstreamer pipeline is a toy example that demonstrates that Gstreamer can parse the mkv passed into it. 

## Release Notes
### Release 1.0.7 (Sep 2018)
* Add flag in KinesisVideoRendererExample and KinesisVideoExample to use the existing stream (and not doing PutMedia again if it exists already).
* Added support to retrieve the information from FragmentMetadata and display in the image panel during rendering.

### Release 1.0.6 (Sep 2018)
* Introduce handling for empty fragment metadata
* Added a new SimpleFrame Visitor to handle video with no tags
* Refactored H264FrameDecoder, so that base class method can be reused by child class

### Release 1.0.5 (May 2018)
* Introduce `GetMediaResponseStreamConsumer` as an abstract class used to consume the output of a GetMedia* call
to Kinesis Video in a streaming fashion. Child classes will use visitors to implement different consumers.
* The `MergedOutputPiper` extends `GetMediaResponseStreamConsumer` to merge consecutive mkv streams in the output of GetMedia
   and pipes the merged stream to the stdin of a child process.
* Add the capability and example to pipe the output of GetMedia calls to GStreamer using `MergedOutputPiper`.

### Release 1.0.4 (April 2018)
* Add example for KinesisVideo Streams integration with Rekognition and draw Bounding Boxes for every sampled frame.
* Fix for stream ending before reporting tags visited.
* Same test data file for parsing and rendering example.
* Known Issues:  In `KinesisVideoRekognitionIntegrationExample`, the decode/renderer sample using JCodec may not be able to decode all mkv files.

### Release 1.0.3 (Februrary 2018)
*  In OutputSegmentMerger, make sure that the lastClusterTimecode is updated for the first fragment.
If timecode is equal to that of a previous cluster, stop merging
* FrameVisitor to process the frames in the output of a GetMedia call.
* CopyVisitor to copy the raw bytes of the stream being parsed to an output stream.
* Add example that shows parsing and rendering Kinesis Video Streams.
* Known Issues:  In `KinesisVideoRendererExample`, the decode/renderer sample using JCodec may not be able to decode all mkv files.
   
### Release 1.0.2 (December 2017)
* Add example that shows integration with Kinesis Video Streams.
* Remove unnecessary import.

### Release 1.0.1 (November 2017)
* Update to include the url for Amazon Kinesis Video Streams in the pom.xml

### Release 1.0.0 (November 2017)
* First release of the Amazon Kinesis Video Parser Library.
* Supports Mkv elements up to version 4. 
* Known issues:
    * EBMLMaxIDLength and EBMLMaxSizeLength are hardcoded as 4 and 8 bytes respectively
    * Unknown EBML elements not specified in `MkvTypeInfos` are not readable by the user using `StreamingMkvReader`.
    * Unknown EBML elements not specified in `MkvTypeInfos` of unknown length lead to an exception.
    * Does not do any CRC validation for any Mkv elements with the `CRC-32` element. 
