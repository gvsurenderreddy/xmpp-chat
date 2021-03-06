# XMPP Chat for live helper chat
This is repository for full XMPP chat support in Live Helper Chat. Standalone and Automated hosting enviroments are supported.
Folders structure
 * ejabberd_mod - contains two ejabberd mods required for chat integration
 * ejabberd_xmlrpc - optional if rpc as handler is used. Recommended way.
 * node - node server used by LHC PHP extension
 * xmppservice - LHC extenstion which have to be installed in lhc itself.

## Requirements
 * https://www.ejabberd.im/ with mod_admin_extra enabled 15.04v
 * https://livehelperchat.com/
 * [Optional] https://nodejs.org
 * [Optional, but recommended] Advanced linux skills. This stuff is not for amateurs!!!

## Workflow how everything is related

### LHC extension requirements
 * Option to create an XMPP users and assign operators to them.
 * It should be able to handle three types of accounts, [chat]/[online visitor]/[operator]
 * Online visitors/chats account should be expirable
 * Admin can select should selected operator receives new chat request's or not.
 * If operator sends a message to chat but chat has already another owner, operator is informed that chat was already accepted
 * If operator writes a message to chat and chat is still pending chat owner becames XMPP sender and his message is send to user.
 * It supports automated hosting also.

### Username constructing patterns in standalone enviroment
 * visitors.[visitor_id]@xmpp.example.com
 * visitors.[chat_id].chat@xmpp.example.com
 * Shared roasters should be constructed like
  * visitors
  * operators

### Visitor comes to site (node server takes care of this API)
 * LHC extension checks does passed online user has assigned XMPP user.
 * If user does not exists create xmpp account

### Optional if node as handler is used
 * Visitor status should be appeneded with online information. It should come from request to NodeJS server.
 * Visitor status should be online for 5 minutes since last page view
 * To nodejs server is provided request to register XMPP account and set it as active
  
### Operator sends a message to online visitor. [mod_lhc]
 * Operator writes message to user using XMPP client.
 * Intercept message at ejabberd lhc extension and there execute callback to lhc module.
 * Mod should send messages only if sender is an operator.
 * If we find active chat [based on assigned chat_id to online visitor] we append message to chat if not we write this as invitation to chat to online visitor.
 
### Visitor starts a chat [nodejs]
 * First we check does chat has assigned online visitor id and if online visitor has a xmpp account record under it.
 * We send message to all online operator who have permission to handle user department and have associated XMPP account.
 * If chat does not have associated online visitor XMPP account we create a new account based on chat itself.
 * We issue request to NodeJS to register this account and then we send message to all operators.
 
### Send a message to operator from visitor (during the chat)
 * Connect as user and send message to assigned operator if it exists.
 * If we cannot find active XMPP account associated with chat we send request to register XMPP account.
 * Once we do this, we just send http request to nodejs api or directly from PHP and send a message to ejabberd. Basically we login as visitor and send message to assigned operator xmpp user.
 
### Send a message from operator to visitor (during the chat)[mod_lhc]
 * We intercept a message and send post request to lhc module, which based on online visitor id, determines a chat and publishes a message to chat.
 * Messages from visitors to operators should be ignored and not send to LHC callbacks
 
### Operators status monitoring [mod_lhcping.erl,mod_lhc.erl]
 * This ejabberd module monitors statuses and sends pings to provided URL in JSON content
 * Callback is send also then operator logins
 * Callback is also send then operator logouts
 * Pings to online visitors should be ignored and pings only for operators should be send
 * mod_lhc also handles offline messages and forwards them to HTTP
 
### Autoamated hosting workflow
 * Each operator in instrance will get username contructed in [username].[instance_name]@xmpp.example.com
 * Each visitor instance will get username constructed like visitor.[visitor_id].[instance_name]@xmpp.example.com
  * [instance_name] cannot be "chat"
 * Each direct chat xmpp username should be constructed like visitor.[chat_id].chat.[instance_name]@xmpp.example.com
 * Each instance will have two shared roasters, they should be constructed following this pattern
  * visitors.[instance_name]
  * operators.[instance_name]
 * Then instance is created extension should precreate these two shared roasters
 * Once instance is destroyed shared roasters and users should be unregistered from ejabberd
