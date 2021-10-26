# Create a subscription

```java
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