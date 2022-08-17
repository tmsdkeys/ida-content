# Adding Packet and Acknowledgement data

In this section we learn how to define packet and acks (acknowledgements) for the Leaderboard blockchain. Remember that this blockchain will mostly be receiving packets from the checkers blockchain or other gaming chains. This will be handled in the checkers blockchain extension tutorial. In this section we will add an additional packet definition that will enable the Leaderboard chain to send a packet to connected game chains when a player has entered the top of the rankings.

The documentation on how to define packet and acks in IBC can be found in [the IBC go docs](https://ibc.cosmos.network/main/ibc/apps/packets_acks.html).

## Scaffold a packet with Ignite CLI

We are now going to be scaffolding the IBC packet data with Ignite CLI and compare once more with git diff:

```bash
ignite scaffold packet ibcTopRank playerId rank score --ack playerId --module leaderboard
```

Note that the packet is called `ibcTopRank`, which includes the fields `playerId`, `rank` and `score`. Additionally we send back the `playerId` of the player who entered the top of the rankings through the `Acknowledgement`.

We see the output on the terminal that gives an overview of the changes made:

```bash
modify proto/leaderboard/packet.proto
modify proto/leaderboard/tx.proto
modify x/leaderboard/client/cli/tx.go
create x/leaderboard/client/cli/tx_ibc_top_rank.go
modify x/leaderboard/handler.go
create x/leaderboard/keeper/ibc_top_rank.go
create x/leaderboard/keeper/msg_server_ibc_top_rank.go
modify x/leaderboard/module_ibc.go
modify x/leaderboard/types/codec.go
modify x/leaderboard/types/events_ibc.go
create x/leaderboard/types/messages_ibc_top_rank.go
create x/leaderboard/types/messages_ibc_top_rank_test.go
create x/leaderboard/types/packet_ibc_top_rank.go

🎉 Created a packet `ibcTopRank`.
```
In the next paragraphs we will investigate each of the most important additions to the code.

## Proto definitions

The first additions are to the proto definitions in the `packet.proto` and `tx.proto` files.

```diff
@@ message LeaderboardPacketData {
    oneof packet {
        NoData noData = 1;
        // this line is used by starport scaffolding # ibc/packet/proto/field
+		IbcTopRankPacketData ibcTopRankPacket = 2; 
        // this line is used by starport scaffolding # ibc/packet/proto/field/number
    }
}
```
with `IbcTopRankPacketData`:
```protobuf
// IbcTopRankPacketData defines a struct for the packet payload
message IbcTopRankPacketData {
  string playerId = 1;
  string rank = 2;
  string score = 3;
}
```
and the ack:
```protobuf
// IbcTopRankPacketAck defines a struct for the packet acknowledgment
message IbcTopRankPacketAck {
	  string playerId = 1;
}
```

And in `tx.proto` a Msg service is added:
```diff
// Msg defines the Msg service.
service Msg {
+      rpc SendIbcTopRank(MsgSendIbcTopRank) returns (MsgSendIbcTopRankResponse);
    // this line is used by starport scaffolding # proto/tx/rpc
}
```

where: 

```protobuf
message MsgSendIbcTopRank {
  string creator = 1;
  string port = 2;
  string channelID = 3;
  uint64 timeoutTimestamp = 4;
  string playerId = 5;
  string rank = 6;
  string score = 7;
}

message MsgSendIbcTopRankResponse {
}
```

**Note**: the proto definitions will be compiled into `types/packet.pb.go` and `types/tx.pb.go`.

## CLI commands

Ignite CLI also creates CLI commands to send packets and adds them to the `client/cli/` folder.

We can thus send packets from the CLI with the following command:
```bash
leaderboardd tx leaderboard send-ibcTopRank [portID] [channelID] [playerId] [rank] [score]
```

## SendPacket and Packet callback logic

When scaffolding an IBC module with Ignite CLI, we already saw the implementation of the `IBCModule` interface including barebones packet callbacks structure. Now that we've also scaffolded a packet (and ack), the callbacks have been added with logic to handle the receive, ack and timeout scenarios. 

Additionally for the sending of a packet, a message server has been added that handles a SendPacket message, in this case `MsgSendIbcTopRank`.

**NOTE**: IBC allows some freedom to the developers how to implement the custom logic, decoding and encoding packets and processing acks. The provided structure is but one example how to tackle this. Therefore it makes sense to focus on the general flow to handle user messages or IBC callbacks rather than the specific implementation by Ignite CLI.

### Sending packets

To handle a user submitting a message to send an IBC packet, a message server is added to the handler.

```diff
    @@ x/leaderboard/handler.go
    func NewHandler(k keeper.Keeper) sdk.Handler {
+        msgServer := keeper.NewMsgServerImpl(k)

        return func(ctx sdk.Context, msg sdk.Msg) (*sdk.Result, error) {
            ctx = ctx.WithEventManager(sdk.NewEventManager())

            switch msg := msg.(type) {
+            case *types.MsgSendIbcTopRank:
+               res, err := msgServer.SendIbcTopRank(sdk.WrapSDKContext(ctx), msg)
+               return sdk.WrapServiceResult(ctx, res, err)
                // this line is used by starport scaffolding # 1
            default:
                errMsg := fmt.Sprintf("unrecognized %s message type: %T", types.ModuleName, msg)
                return nil, sdkerrors.Wrap(sdkerrors.ErrUnknownRequest, errMsg)
            }
        }
    }
```
It calls the `SendIbcTopRank` method, defined as:
```go
func (k msgServer) SendIbcTopRank(goCtx context.Context, msg *types.MsgSendIbcTopRank) (*types.MsgSendIbcTopRankResponse, error) {
	ctx := sdk.UnwrapSDKContext(goCtx)

	// TODO: logic before transmitting the packet

	// Construct the packet
	var packet types.IbcTopRankPacketData

	packet.PlayerId = msg.PlayerId
	packet.Rank = msg.Rank
	packet.Score = msg.Score

	// Transmit the packet
	err := k.TransmitIbcTopRankPacket(
		ctx,
		packet,
		msg.Port,
		msg.ChannelID,
		clienttypes.ZeroHeight(),
		msg.TimeoutTimestamp,
	)
	if err != nil {
		return nil, err
	}

	return &types.MsgSendIbcTopRankResponse{}, nil
}
```
Which in turn calls the `TransmitIbcTopRankPacket` method. This method gets all of the required metadata from core IBC before sending the packet using the ChannelKeeper's `SendPacket` function.

```go
if err := k.ChannelKeeper.SendPacket(ctx, channelCap, packet);
    err  != nil {
		return err
	}
```

Note that when we want to add additional custom logic before transmitting the packet, we do this in the `SendIbcTopRank` method on the message server.

### Receiving packets

In a previous section we already examined the `OnRecvPacket` callback in the `x/leaderboard/module_ibc.go` file. 

There Ignite CLI had setup a structure to dispatch the packet depending on packet type through a switch statement. Now by adding the `IbcTopRank` packet, a case has been added:

```go
// @ switch packet := modulePacketData.Packet.(type) in OnRecvPacket
case *types.LeaderboardPacketData_IbcTopRankPacket:
		packetAck, err := am.keeper.OnRecvIbcTopRankPacket(ctx, modulePacket, *packet.IbcTopRankPacket)
		if err != nil {
			ack = channeltypes.NewErrorAcknowledgement(err.Error())
		} else {
			// Encode packet acknowledgment
			packetAckBytes, err := types.ModuleCdc.MarshalJSON(&packetAck)
			if err != nil {
				return channeltypes.NewErrorAcknowledgement(sdkerrors.Wrap(sdkerrors.ErrJSONMarshal, err.Error()).Error())
			}
			ack = channeltypes.NewResultAcknowledgement(sdk.MustSortJSON(packetAckBytes))
		}
		ctx.EventManager().EmitEvent(
			sdk.NewEvent(
				types.EventTypeIbcTopRankPacket,
				sdk.NewAttribute(sdk.AttributeKeyModule, types.ModuleName),
				sdk.NewAttribute(types.AttributeKeyAckSuccess, fmt.Sprintf("%t", err != nil)),
			),
		)
```

The first line of code in the case statement calls the application's `OnRecvPacket` callback on the keeper to process the reception of the packet. 

```go
// OnRecvIbcTopRankPacket processes packet reception
func (k Keeper) OnRecvIbcTopRankPacket(ctx sdk.Context, packet channeltypes.Packet, data types.IbcTopRankPacketData) (packetAck types.IbcTopRankPacketAck, err error) {
	// validate packet data upon receiving
	if err := data.ValidateBasic(); err != nil {
		return packetAck, err
	}

	// TODO: packet reception logic

	return packetAck, nil
}
```

Remember that the `OnRecvPacket` callback writes an acknowledgement as well (we cover the synchronous write ack case).

### Acknowledging packets

Similarly to the `OnRecvPacket` case before, Ignite CLI already had prepared the structure of the `OnAcknowledgementPacket` with the switch statement. Again scaffolding the packet adds a case to the switch.

```go
// @ switch packet := modulePacketData.Packet.(type) in OnAcknowledgmentPacket
	case *types.LeaderboardPacketData_IbcTopRankPacket:
		err := am.keeper.OnAcknowledgementIbcTopRankPacket(ctx, modulePacket, *packet.IbcTopRankPacket, ack)
		if err != nil {
			return err
		}
		eventType = types.EventTypeIbcTopRankPacket
```

Which calls into the newly created application keeper's ack packet callback:
```go
func (k Keeper) OnAcknowledgementIbcTopRankPacket(ctx sdk.Context, packet channeltypes.Packet, data types.IbcTopRankPacketData, ack channeltypes.Acknowledgement) error {
	switch dispatchedAck := ack.Response.(type) {
	case *channeltypes.Acknowledgement_Error:

		// TODO: failed acknowledgement logic
		_ = dispatchedAck.Error

		return nil
	case *channeltypes.Acknowledgement_Result:
		// Decode the packet acknowledgment
		var packetAck types.IbcTopRankPacketAck

		if err := types.ModuleCdc.UnmarshalJSON(dispatchedAck.Result, &packetAck); err != nil {
			// The counter-party module doesn't implement the correct acknowledgment format
			return errors.New("cannot unmarshal acknowledgment")
		}

		// TODO: successful acknowledgement logic

		return nil
	default:
		// The counter-party module doesn't implement the correct acknowledgment format
		return errors.New("invalid acknowledgment format")
	}
}
```
It allows to add custom application logic for both failed and successfull acks.

### Timing out packets

Timing out the packets follows the same flow, adding a case to the switch statement in `OnTimeoutPacket`, calling into the keeper's timeout packet callback where the custom logic can be implemented.

We leave it up to the reader to check this out.

### Extra bits

Next to the above, also some additions have been made to the `types` package. These include `codec.go`, `events_ibc.go`, `messages_ibc_top_rank.go`.


