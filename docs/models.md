# ATLAS Advanced Streaming Samples

![Build Status](https://mat-ocs.visualstudio.com/Telemetry%20Analytics%20Platform/_apis/build/status/MAT.OCS.Streaming/Streaming%20Samples?branchName=develop)

Table of Contents
=================
<!--ts-->
* [Introduction - MAT.OCS.Streaming library](/README.md)
* [C# Samples](/README.md)
* [Model sample](/docs/models.md)
<!--te-->

# Model sample

A basic example of a model calculating the total horizontal acceleration parameter called gTotal. 

gTotal = |gLat| + |gLong|. 

Model consists from two classes, [ModelSample](https://github.com/McLarenAppliedTechnologies/mat.ocs.streaming.samples/blob/enh/gTotal/src/MAT.OCS.Streaming.Samples/Samples/Models/ModelSample.cs) and [StreamModel](https://github.com/McLarenAppliedTechnologies/mat.ocs.streaming.samples/blob/enh/gTotal/src/MAT.OCS.Streaming.Samples/Samples/Models/StreamModel.cs)
.
## Environment setup
You need to prepare environment variables as follows:

[Environment variables](https://github.com/McLarenAppliedTechnologies/mat.ocs.streaming.samples/blob/enh/gTotal/src/MAT.OCS.Streaming.Samples/Samples/Models/ModelSample.cs#L17-L20)
```cs
private const string DependencyUrl = "http://10.228.4.9:8180/api/dependencies/";
private const string InputTopicName = "ModelsInput";
private const string OutputTopicName = "ModelsOutput";
private const string BrokerList = "10.228.4.22";
```

## Output format 
Specify output data of the model and publish it to dependency service:

[Output format](https://github.com/McLarenAppliedTechnologies/mat.ocs.streaming.samples/blob/enh/gTotal/src/MAT.OCS.Streaming.Samples/Samples/Models/ModelSample.cs#L37-L47)
```cs
var outputDataFormat = DataFormat.DefineFeed()
                            .Parameters(new List<string> { "gTotal:vTag" })
                            .AtFrequency(100)
                            .BuildFormat();

this.dataFormatId = dataFormatClient.PutAndIdentifyDataFormat(outputDataFormat);

var acClient = new AtlasConfigurationClient(dependencyClient);

var atlasConfiguration = this.CreateAtlasConfiguration();
this.atlasConfId = acClient.PutAndIdentifyAtlasConfiguration(atlasConfiguration);
```
## Subscribe
[Subscribe](https://github.com/McLarenAppliedTechnologies/mat.ocs.streaming.samples/blob/enh/gTotal/src/MAT.OCS.Streaming.Samples/Samples/Models/ModelSample.cs#L49-L58) for input topic streams:

```cs
using (var client = new KafkaStreamClient(BrokerList))
{
    using (var outputTopic = client.OpenOutputTopic(OutputTopicName))
    using (var pipeline = client.StreamTopic(InputTopicName).Into(streamId => this.CreateStreamPipeline(streamId, outputTopic)))
    {
        cancellationToken.WaitHandle.WaitOne();
        pipeline.Drain();
        pipeline.WaitUntilStopped(new TimeSpan(0, 0, 1), default(CancellationToken));
    }
}
```

## Into
Each stream will raise callback [Into](https://github.com/McLarenAppliedTechnologies/mat.ocs.streaming.samples/blob/enh/gTotal/src/MAT.OCS.Streaming.Samples/Samples/Models/ModelSample.cs#L61-L66)() where a new instance of the model for the new stream is created.

This block of code is called for each new stream. 

```cs
private IStreamInput CreateStreamPipeline(string streamId, IOutputTopic outputTopic)
{
    var streamModel = new StreamModel(this.dataFormatClient, outputTopic, this.dataFormatId, this.atlasConfId);

    return streamModel.CreateStreamInput(streamId);
}
```

## Stream model
For each new stream, we create a instance of StreamModel class.
[StreamModel](https://github.com/McLarenAppliedTechnologies/mat.ocs.streaming.samples/blob/enh/gTotal/src/MAT.OCS.Streaming.Samples/Samples/Models/StreamModel.cs)

### Create stream input
At the beginning of each stream, we create new stream input and output.

[CreateStreamInput](https://github.com/McLarenAppliedTechnologies/mat.ocs.streaming.samples/blob/enh/gTotal/src/MAT.OCS.Streaming.Samples/Samples/Models/StreamModel.cs#L25-L51)

```cs
public IStreamInput CreateStreamInput(string streamId)
{
    // these templates provide commonly-combined data, but you can make your own
    var input = new SessionTelemetryDataInput(streamId, dataFormatClient);
    var output = new SessionTelemetryDataOutput(outputTopic, this.outputDataFormatId, dataFormatClient);

    this.outputFeed = output.DataOutput.BindFeed("");
    
    // we add output format reference to output session.
    output.SessionOutput.AddSessionDependency(DependencyTypes.DataFormat, this.outputDataFormatId);
    output.SessionOutput.AddSessionDependency(DependencyTypes.AtlasConfiguration, this.outputAtlasConfId);

    // automatically propagate session metadata and lifecycle
    input.LinkToOutput(output.SessionOutput, identifier => identifier + "_Models");

    // we simply formward laps.
    input.LapsInput.LapStarted += (s, e) => output.LapsOutput.SendLap(e.Lap);

    // we bind our models to specific feed and parameters.
    input.DataInput.BindDefaultFeed("gLat:Chassis", "gLong:Chassis").DataBuffered += this.gTotalModel;

    return input;
}
```
### gTotal function
In the callback, each bucket of data is calculated and the result is sent to the output topic.
[gTotal ](https://github.com/McLarenAppliedTechnologies/mat.ocs.streaming.samples/blob/enh/gTotal/src/MAT.OCS.Streaming.Samples/Samples/Models/StreamModel.cs#L53-L85)

```cs
private void gTotalModel(object sender, TelemetryDataFeedEventArgs e)
{
    var inputData = e.Buffer.GetData();
    var data = new TelemetryData();

    data.EpochNanos = inputData.EpochNanos;
    data.Parameters = new TelemetryParameterData[1];
    data.TimestampsNanos = inputData.TimestampsNanos;

    data.Parameters = new TelemetryParameterData[1];
    data.Parameters[0] = new TelemetryParameterData();

    data.Parameters[0].AvgValues = new double[inputData.TimestampsNanos.Length];
    data.Parameters[0].Statuses = new DataStatus[inputData.TimestampsNanos.Length];

    for (var index = 0; index < inputData.TimestampsNanos.Length; index++)
    {
        var gLat = inputData.Parameters[0].AvgValues[index];
        var gLong = inputData.Parameters[1].AvgValues[index];

        var gLatStatus = inputData.Parameters[0].Statuses[index];
        var gLongStatus = inputData.Parameters[1].Statuses[index];

        data.Parameters[0].AvgValues[index] = Math.Abs(gLat) + Math.Abs(gLong);
        data.Parameters[0].Statuses[index] = (gLatStatus & DataStatus.Sample) > 0 && (gLongStatus & DataStatus.Sample) > 0
                                                ? DataStatus.Sample
                                                : DataStatus.Missing;

    }

    outputFeed.EnqueueAndSendData(data);
}
```