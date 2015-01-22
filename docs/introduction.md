Introduction
=====

Donc vous cherchez une librarie de Messaging, vous etes frustré par WCF et MSMQ (on était dans le même cas que vous), vous avez entendu dire que ZeroMq était très rapide et maintenant vous êtes là, sur NetMq, le portage .Net de ZeroMQ.

Alors oui, NetMQ est une librairie de messaging et oui c'est rapide... Mais NetMQ nécessite un peu d'apprentissage et nous allons vous aider.


## Par où commencer

ZeroMQ et netMQ ne sont pas juste des librairies qu'il suffit de télécharger, regarder quelques exemples de code et voilà...
Il y a une véritable philosophie derrière pour apprendre à s'en servir correctement et bien comprendre comment celà fonctionne. La meilleure façon de commencer est de lire le <a href="http://zguide.zeromq.org/page:all" target="_blank">ZeroMQ guide</a>. lisez le, re-lisez et revenez ensuite.

## Le Zero de ZeroMQ

La philosophie de ZeroMQ commence avec le Zero. Zero pour zero fournisseur, zero latence, zero coût (c'est gratuit !), zero administration.

Plus généralement, "zero" refère à la culture du minimalisme (C'est ce qui conduit ce projet). Nous rendons NetMQ plus puissant en réduisant la complexité plutôt qu'en ajoutant des couches de fonctionalités.

## Récupérer la librarie

Vous pouvez récupérer la librairie sur <a href="https://nuget.org/packages/NetMQ/" target="_blank">nuget</a>


## Contexte

Le <code>NetMQContext</code> sert à créer TOUTES les Sockets. Tout code NetMQ doit commencer par créer un <code>NetMQContext</code>, ce qui est fait en utilisant la méthode <code>NetMQContext.Create()</code>. <code>NetMQContext</code> est <code>IDisposable</code> et donc utilisable dans un bloc <code>using</code>.

Vous ne devez créer et utiliser qu'un seul contexte par process. Techniquement, le contexte est le conteneur de toutes les sockets pour un process et agit comme le connecteur pour les sockets de type "inproc", ce qui est la manière la plus rapide de connecter des thread dans un process. Si au runtime, un process a deux contextes, c'est comme ci vous aviez deux instances de NetMQ. Si c'est explicitement ce que vous voulez, Ok, mais sinon suivez cette rêgle : 

N'ayez qu'UN <code>NetMQContext</code>. Vous pouvez créer toutes les sockets avec dans le process.


## Recevoir et Envoyer

NetMQ n'utilise ques des socket, il est donc normal de pouvoir envoyer et recevoir des données. 
Une page dédiée est disponible ici : <a href="http://netmq.readthedocs.org/en/latest/receiving-sending/">Recevoir et Envoyer</a>.



## Premier Exemple

Le classique "Hello world" (naturellement).

**Server:**
    
    static void Main(string[] args)
    {
        using (var context = NetMQContext.Create())
        {
            using (var server = context.CreateResponseSocket())
            {
                server.Bind("tcp://*:5555");

                while (true)
                {
                    var message = server.ReceiveString();

                    Console.WriteLine("Received {0}", message);

                    // processing the request
                    Thread.Sleep(100);

                    Console.WriteLine("Sending World");
                    server.Send("World");
                }
            }
        }
    }
 
Le serveur crée une socket de type "response" (vous pouvez en savoir plus dans le chapitre request-response), la bind au port 5555 et attend des messages. Il n'y a pas de configuration. Le server renvoi une String. NetMQ peut renvoyer autre chose que des String, mais il ne fait pas automatiquement de sérialization. Vous devez le faire vous-même mais vous verrez plus tard quelques petites astuces (les messages Multi Part).
  
**Client:**
    
    static void Main(string[] args)
    {
        using (var context = NetMQContext.Create())
        {
            using (var client = context.CreateRequestSocket())
            {
                client.Connect("tcp://localhost:5555");

                for (int i = 0; i < 10; i++)
                {
                    Console.WriteLine("Sending Hello");
                    client.Send("Hello");
                        
                    var message = client.ReceiveString();                        
                    Console.WriteLine("Received {0}", message);
                }
            }
        }
    }

Le client créée une socket de type "request", se connecte et commence a envoyer des messages.

Les méthodes d'envoi et de réception sont bloquantes. Pour la réception, c'est simple : s'il n y a pas de messages, on attend.
Pour l'envoi, c'est plus compliqué et celà dépend du type de socket. Dans une socket de type "request", la méthode bloque jusqu'a ce que toute les données soit envoyées. Si aucune socket "response" n'est connectée, la méthode bloque jusqu'a ce qu'une socket recoive le message et renvoi une réponse.

Néanmoins vous pouvez appeller ces méthodes avec le paramêtre <code>DontWait</code> pour ne pas attendre. N'oubliez pas d'encapsuler l'appel par un try-catch sur l'exception <code>AgainException</code>, comme celà :

    try
    {
        var message = client.ReceiveString(SendReceiveOptions.DontWait);
        Console.WriteLine("Received {0}", message);
    }
    catch (AgainException ex)
    {
        Console.WriteLine(ex);                        
    }


## Bind Versus Connect

Dans l'exemple précédent, vous avez surement remarqué que le *Server* utilise *Bind* alors que le *Client* utilise *Connect*. Pourquoi ? Quelle est la différence ?

ZeroMQ crée une file d'attente de message (Queue) par connection. Exemple, si votre socket est connectés à trois autres sockets, il y a 3 files d'attentes de message.

Avec Bind, vous autorisez les autres socket à se connecter à vous. Comme vous ne pouvez pas savoir combien de socket vont se connecter dans le futur, vous ne pouvez pas créer ces files d'attentes en avance. Elles sont donc créées à chaque fois qu'une socket clientes se connecte à vous.

Avec Connect, ZeroMQ sait qu'une seule files d'attentes est à créer par connection. Ceci s'applique sur tous les type de socket à l'exception de la socket de type "ROUTER". Dans ce cas, les files d'attentes ne sont créées qu'après avoir reçus un accusé de connection de la part de la socket ou l'on se connect.

Donc envoyer un message sur une socket ou personnes n'est lié (bind) où sur une socket ROUTER ou aucune connection n'est active ne crée pas de file d'attente pour stocker les messages.

Quand dois je utiliser Bind et quand dois je utiliser Connect ?

En rêgle générale : 
As a very general advice: use bind on the most stable points in your architecture and connect from the more volatile endpoints. For request/reply the service provider might be point where you bind and the client uses connect. Like plain old TCP.

If you can't figure out which parts are more stable (i.e. peer-to-peer) think about a stable device in the middle, where boths sides can connect to.

You can read more about this at the ZeroMQ FAQ http://zeromq.org/area:faq under the "Why do I see different behavior when I bind a socket versus connect a socket?" section. 




## Multipart messages

ZeroMQ/NetMQ work on the concept of frames, as such most messages are considered to be made up of one or more frames. NetMQ provides some convenience methods to allow you to send string messages. You should however, familiarise yourself with the the idea of multiple frames and how they work.

This is covered in much more detail in the [Message](http://netmq.readthedocs.org/en/latest/message/) documentation page




## Patterns

<a href="http://zguide.zeromq.org/page:all" target="_blank">ZeroMQ guide</a> (and therefore NetMQ) is all about patterns, and building blocks. The <a href="http://zguide.zeromq.org/page:all" target="_blank">ZeroMQ guide</a> covers everything you need to know to help you with these patterns. You should make sure you read the following sections before you attempt to start work with NetMQ.

+ <a href="http://zguide.zeromq.org/page:all#Chapter-Sockets-and-Patterns" target="_blank">Chapter 2 - Sockets and Patterns</a>
+ <a href="http://zguide.zeromq.org/page:all#Chapter-Advanced-Request-Reply-Patterns" target="_blank">Chapter 3 - Advanced Request-Reply Patterns</a>
+ <a href="http://zguide.zeromq.org/page:all#Chapter-Reliable-Request-Reply-Patterns" target="_blank">Chapter 4 - Reliable Request-Reply Patterns</a>
+ <a href="http://zguide.zeromq.org/page:all#Chapter-Advanced-Pub-Sub-Patterns" target="_blank">Chapter 5 - Advanced Pub-Sub Patterns</a>


NetMQ also has some examples of a few of these patterns written using the NetMQ APIs. Should you find the pattern you are looking for in the <a href="http://zguide.zeromq.org/page:all" target="_blank">ZeroMQ guide</a> it should be fairly easy to translate that into NetMQ usage.

Here are some links to the patterns that are available within the NetMQ codebase:

+ <a href="https://github.com/zeromq/netmq/tree/master/src/Samples/Brokerless%20Reliability%20(Freelance%20Pattern)/Model%20One" target="_blank">Brokerless Reliability Pattern - Freelance Model one</a>
+ <a href="https://github.com/zeromq/netmq/tree/master/src/Samples/Load%20Balancing%20Pattern" target="_blank">Load Balancer Patterns</a>
+ <a href="https://github.com/zeromq/netmq/tree/master/src/Samples/Pirate%20Pattern/Lazy%20Pirat" target="_blank">Lazy Pirate Pattern</a>
+ <a href="https://github.com/zeromq/netmq/tree/master/src/Samples/Pirate%20Pattern/Simple%20Pirate" target="_blank">Simple Pirate Pattern</a>



For other patterns, the <a href="http://zguide.zeromq.org/page:all" target="_blank">ZeroMQ guide</a> will be your first port of call

ZeroMQ patterns are implemented by pairs of sockets with matching types. In other words, to understand ZeroMQ patterns you need to understand socket types and how they work together. Mostly, this just takes study; there is little that is obvious at this level.

The built-in core ZeroMQ patterns are:

+ **Request-reply**, which connects a set of clients to a set of services. This is a remote procedure call and task distribution pattern.
+ **Pub-sub**, which connects a set of publishers to a set of subscribers. This is a data distribution pattern.
+ **Pipeline**, which connects nodes in a fan-out/fan-in pattern that can have multiple steps and loops. This is a parallel task distribution and collection pattern.
+ **Exclusive pair**, which connects two sockets exclusively. This is a pattern for connecting two threads in a process, not to be confused with "normal" pairs of sockets.

These are the socket combinations that are valid for a connect-bind pair (either side can bind):

+ <code>PublisherSocket</code> and <code>SubscriberSocket</code>
+ <code>RequestSocket</code> and <code>ResponseSocket</code>
+ <code>RequestSocket</code>  and <code>RouterSocket</code>
+ <code>DealerSocket</code> and <code>ResponseSocket</code>
+ <code>DealerSocket</code> and <code>RouterSocket</code>
+ <code>DealerSocket</code> and <code>DealerSocket</code>
+ <code>RouterSocket</code> and <code>RouterSocket</code>
+ <code>PushSocket</code> and <code>PullSocket</code>
+ <code>PairSocket</code> and <code>PairSocket</code>

Any other combination will produce undocumented and unreliable results, and future versions of ZeroMQ will probably return errors if you try them. You can and will, of course, bridge other socket types via code, i.e., read from one socket type and write to another.




## Options

NetMQ comes with several options that will effect how things work. 

Depending on the type of sockets you are using, or the topology you are attempting to create, you may find that you need to set some NeroMQ options. In NetMQ this is done using the xxxxSocket.Options property.

Here is a listing of the available properties that you may set on a xxxxSocket. It is hard to say exactly which of these values you may need to set, as that obviously depends entirely on what you are trying to achieve. All I can do is list the options, and make you aware of them. So here they are

+ Affinity  
+ BackLog  
+ CopyMessages
+ DelayAttachOnConnect
+ Endian
+ GetLastEndpoint
+ IPv4Only
+ Identity
+ Linger
+ MaxMsgSize
+ MulticastHops
+ MulticastRate
+ MulticastRecoveryInterval
+ ReceiveHighWaterMark
+ ReceiveMore
+ ReceiveTimeout
+ ReceiveBuffer
+ ReconnectInterval
+ ReconnectIntervalMax
+ SendHighWaterMark
+ SendTimeout
+ SendBuffer
+ TcpAcceptFilter
+ TcpKeepAlive
+ TcpKeepaliveCnt
+ TcpKeepaliveIdle
+ TcpKeepaliveInterval
+ XPubVerbos

We will not be covering all of these here, but shall instead cover them in the areas where they are used. For now just be aware that if you have read something in the <a href="http://zguide.zeromq.org/page:all" target="_blank">ZeroMQ guide</a> that mentions some option, that this is most likely the place you will need to set it/read from it


