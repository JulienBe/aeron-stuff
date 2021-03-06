# Create a subscription

```java
private final SubscriptionMessageFlyweight subscriptionMessage = new SubscriptionMessageFlyweight();
private final RingBuffer toDriverCommandBuffer;


/** Create a proxy to a media driver which sends commands via a {@link RingBuffer}.
 * @param toDriverCommandBuffer to send commands via.
 * @param clientId              to represent the client.
 */
public DriverProxy(final RingBuffer toDriverCommandBuffer, final long clientId) {
    this.toDriverCommandBuffer = toDriverCommandBuffer;
    this.clientId = clientId;
}

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

# Create publication Image


```java
void onCreatePublicationImage(int sessionId, int streamId, int initialTermId, int activeTermId, int initialTermOffset, int termBufferLength, int senderMtuLength, int transportIndex, InetSocketAddress controlAddress, InetSocketAddress sourceAddress, ReceiveChannelEndpoint channelEndpoint) {
    Configuration.validateMtuLength(senderMtuLength);

    UdpChannel subscriptionChannel = channelEndpoint.subscriptionUdpChannel();
    Configuration.validateInitialWindowLength(subscriptionChannel.receiverWindowLengthOrDefault(ctx.initialWindowLength()), senderMtuLength);

    long joinPosition = computePosition(activeTermId, initialTermOffset, LogBufferDescriptor.positionBitsToShift(termBufferLength), initialTermId);
    ArrayList<SubscriberPosition> subscriberPositions = createSubscriberPositions(sessionId, streamId, channelEndpoint, joinPosition);

    if (subscriberPositions.size() > 0) {
        RawLog rawLog = null;
        CongestionControl congestionControl = null;
        UnsafeBufferPosition hwmPos = null;
        UnsafeBufferPosition rcvPos = null;
        try {
            long registrationId = toDriverCommands.nextCorrelationId();
            rawLog = newPublicationImageLog(sessionId, streamId, initialTermId, termBufferLength, isOldestSubscriptionSparse(subscriberPositions), senderMtuLength, registrationId);

            congestionControl = ctx.congestionControlSupplier().newInstance(
                    registrationId, subscriptionChannel, streamId, sessionId, termBufferLength, senderMtuLength, controlAddress, sourceAddress, 
                    ctx.receiverCachedNanoClock(), ctx, countersManager);

            SubscriptionLink subscription = subscriberPositions.get(0).subscription();
            String uri = subscription.channel();
            hwmPos = ReceiverHwm.allocate(tempBuffer, countersManager, registrationId, sessionId, streamId, uri);
            rcvPos = ReceiverPos.allocate(tempBuffer, countersManager, registrationId, sessionId, streamId, uri);

            boolean treatAsMulticast = subscription.group() == INFER ? channelEndpoint.udpChannel().isMulticast() : subscription.group() == FORCE_TRUE;

            PublicationImage image = new PublicationImage(
                    registrationId, ctx, channelEndpoint, transportIndex, controlAddress, sessionId, streamId, initialTermId, activeTermId, initialTermOffset, rawLog, 
                    treatAsMulticast ? ctx.multicastFeedbackDelayGenerator() : ctx.unicastFeedbackDelayGenerator(), subscriberPositions, hwmPos, rcvPos, sourceAddress, congestionControl);

            publicationImages.add(image);
            receiverProxy.newPublicationImage(channelEndpoint, image);

            String sourceIdentity = Configuration.sourceIdentity(sourceAddress);
            for (final SubscriberPosition pos : subscriberPositions) {
                pos.addLink(image);
                clientProxy.onAvailableImage(registrationId, streamId, sessionId, pos.subscription().registrationId(), pos.positionCounterId(), rawLog.fileName(), sourceIdentity);
            }
        } catch (final Exception ex) {
            subscriberPositions.forEach((subscriberPosition) -> subscriberPosition.position().close());
            CloseHelper.quietCloseAll(rawLog, congestionControl, hwmPos, rcvPos);
            throw ex;
        }
    }
}
```
