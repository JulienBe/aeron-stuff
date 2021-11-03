# Create a subscription

```java
private final SubscriptionMessageFlyweight subscriptionMessage = new SubscriptionMessageFlyweight();
private final RingBuffer toDriverCommandBuffer;

public long addSubscription(final String channel, final int streamId) {
  final long registrationId = Aeron.NULL_VALUE;
  final long correlationId = toDriverCommandBuffer.nextCorrelationId(); // the correlation id for the command
  final int length = SubscriptionMessageFlyweight.computeLength(channel.length());
  final int index = toDriverCommandBuffer.tryClaim(ADD_SUBSCRIPTION, length);
  if (index < 0) {
    throw new AeronException("could not write add subscription command");
  }
  subscriptionMessage
    .wrap(toDriverCommandBuffer.buffer(), index)
    .registrationCorrelationId(registrationId)
    .streamId(streamId)
    .channel(channel)
    .clientId(clientId)
    .correlationId(correlationId);
  toDriverCommandBuffer.commit(index);
  return correlationId;
}
```

```
SubscriptionMessageFlyweight
Control message for adding or removing a subscription.
     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                          Client ID                            |
    |                                                               |
    +---------------------------------------------------------------+
    |                    Command Correlation ID                     |
    |                                                               |
    +---------------------------------------------------------------+
    |                 Registration Correlation ID                   |
    |                                                               |
    +---------------------------------------------------------------+
    |                         Stream Id                             |
    +---------------------------------------------------------------+
    |                       Channel Length                          |
    +---------------------------------------------------------------+
    |                       Channel (ASCII)                        ...
   ...                                                              |
    +---------------------------------------------------------------+
*/    
```

In ClientCommanderAdapter

```java
subscriptionMsgFlyweight.wrap(buffer, index);
subscriptionMsgFlyweight.validateLength(msgTypeId, length);

correlationId = subscriptionMsgFlyweight.correlationId();
final long clientId = subscriptionMsgFlyweight.clientId();
final int streamId = subscriptionMsgFlyweight.streamId();
final String channel = subscriptionMsgFlyweight.channel();

if (channel.startsWith(IPC_CHANNEL)) {
    conductor.onAddIpcSubscription(channel, streamId, correlationId, clientId);
} else if (channel.startsWith(SPY_QUALIFIER)) {
    conductor.onAddSpySubscription(channel, streamId, correlationId, clientId);
} else {
    conductor.onAddNetworkSubscription(channel, streamId, correlationId, clientId);
}
```

In DriverConductor

```java
void onAddNetworkSubscription(final String channel, final int streamId, final long registrationId, final long clientId) {
    final UdpChannel udpChannel = UdpChannel.parse(channel, nameResolver);

    validateControlForSubscription(udpChannel);
    validateTimestampConfiguration(udpChannel);

    final SubscriptionParams params = SubscriptionParams.getSubscriptionParams(udpChannel.channelUri(), ctx);

    checkForClashingSubscription(params, udpChannel, streamId);
    final ReceiveChannelEndpoint channelEndpoint = getOrCreateReceiveChannelEndpoint(params, udpChannel, registrationId);

    if (params.hasSessionId) {
        if (1 == channelEndpoint.incRefToStreamAndSession(streamId, params.sessionId)) {
            receiverProxy.addSubscription(channelEndpoint, streamId, params.sessionId);
        }
    } else {
        if (1 == channelEndpoint.incRefToStream(streamId)) {
            receiverProxy.addSubscription(channelEndpoint, streamId);
        }
    }

    final AeronClient client = getOrAddClient(clientId);
    final NetworkSubscriptionLink subscription = new NetworkSubscriptionLink(registrationId, channelEndpoint, streamId, channel, client, params);

    subscriptionLinks.add(subscription);
    clientProxy.onSubscriptionReady(registrationId, channelEndpoint.statusIndicatorCounter().id());

    linkMatchingImages(subscription);
}
```

A few levels down, in DataPacketDispatcher

```java

/**
 * Add a subscription to a channel for given stream and session ids.
 *
 * @param streamId  to capture within a channel.
 * @param sessionId to capture within a stream id.
 */
public void addSubscription(final int streamId, final int sessionId) {
    StreamInterest streamInterest = streamInterestByIdMap.get(streamId);
    if (null == streamInterest) {
        streamInterest = new StreamInterest(false);
        streamInterestByIdMap.put(streamId, streamInterest);
    }

    streamInterest.subscribedSessionIds.add(sessionId);

    final SessionInterest sessionInterest = streamInterest.sessionInterestByIdMap.get(sessionId);
    if (null != sessionInterest && NO_INTEREST == sessionInterest.state) {
        streamInterest.sessionInterestByIdMap.remove(sessionId);
    }
}
```

# RTT measurement

in DataPacketDispatcher

```java

/**
 * Dispatch an RTT measurement message to registered interest.
 *
 * @param channelEndpoint of reception.
 * @param msg             flyweight over the network packet.
 * @param srcAddress      the message came from.
 * @param transportIndex  on which the message was received.
 */
public void onRttMeasurement(ReceiveChannelEndpoint channelEndpoint, RttMeasurementFlyweight msg, InetSocketAddress srcAddress, int transportIndex) {
    final int streamId = msg.streamId();
    final StreamInterest streamInterest = streamInterestByIdMap.get(streamId);
    if (null != streamInterest) {
        int sessionId = msg.sessionId();
        SessionInterest sessionInterest = streamInterest.sessionInterestByIdMap.get(sessionId);
        if (null != sessionInterest && null != sessionInterest.image) {
            if (RttMeasurementFlyweight.REPLY_FLAG == (msg.flags() & RttMeasurementFlyweight.REPLY_FLAG)) {
                InetSocketAddress controlAddress = channelEndpoint.isMulticast(transportIndex) ?
                        channelEndpoint.udpChannel(transportIndex).remoteControl() : srcAddress;
                channelEndpoint.sendRttMeasurement(transportIndex, controlAddress, sessionId, streamId, msg.echoTimestampNs(), 0, false);
            } else {
                sessionInterest.image.onRttMeasurement(msg, transportIndex, srcAddress);
            }
        }
    }
}
```
