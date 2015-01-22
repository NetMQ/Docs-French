Introduction
=====

Donc vous cherchez une librairie de Messaging, vous êtes frustré par WCF et MSMQ (on était dans le même cas que vous), vous avez entendu dire que ZeroMq était très rapide et maintenant vous êtes là, sur NetMq, le portage .Net de ZeroMQ.

Alors oui, NetMQ est une librairie de messaging et oui c'est rapide... Mais NetMQ nécessite un peu d'apprentissage et nous allons vous aider.


## Par où commencer

ZeroMQ et netMQ ne sont pas juste des librairies qu'il suffit de télécharger et regarder quelques exemples de code...
Il y a une véritable philosophie derrière pour apprendre à s'en servir correctement et bien comprendre comment celà fonctionne. La meilleure façon de commencer est de lire le <a href="http://zguide.zeromq.org/page:all" target="_blank">ZeroMQ guide</a>. lisez le, re-lisez et revenez ensuite.

## Le Zero de ZeroMQ

La philosophie de ZeroMQ commence avec le Zero. Zero pour zero fournisseur, zero latence, zero coût (c'est gratuit !), zero administration.

Plus généralement, "zero" refère à la culture du minimalisme (C'est ce qui conduit ce projet). Nous rendons NetMQ plus puissant en réduisant la compléxité plutôt qu'en ajoutant des couches de fonctionnalités.

## Récupérer la librarie

Vous pouvez récupérer la librairie sur <a href="https://nuget.org/packages/NetMQ/" target="_blank">nuget</a>


## Contexte

Le <code>NetMQContext</code> sert à créer TOUTES les Sockets. Tout code NetMQ doit commencer par créer un <code>NetMQContext</code>, ce qui est fait en utilisant la méthode <code>NetMQContext.Create()</code>. <code>NetMQContext</code> est <code>IDisposable</code> et donc utilisable dans un bloc <code>using</code>.

Vous ne devez créer et utiliser qu'un seul contexte par process. Techniquement, le contexte est le conteneur de toutes les sockets pour un process et agit comme le connecteur pour les sockets de type "inproc", ce qui est la manière la plus rapide de connecter des thread dans un process. Si au runtime, un process a deux contextes, c'est comme ci vous aviez deux instances de NetMQ. Si c'est explicitement ce que vous voulez, Ok, mais sinon suivez cette rêgle : 

N'ayez qu'UN <code>NetMQContext</code>. Vous pouvez créer toutes les sockets avec dans le process.


## Recevoir et Envoyer

NetMQ n'utilise ques des socket, il est donc normal de pouvoir envoyer et recevoir des données. 
Une page dédiée est disponible ici : <a href="http://netmq.readthedocs.org/fr/latest/receiving-sending/">Recevoir et Envoyer</a>.



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

Avec Connect, ZeroMQ sait qu'une seule file d'attente est à créer par connection. Ceci s'applique sur tous les types de socket à l'exception de la socket "ROUTER". Dans ce cas, les files d'attentes ne sont créées qu'après avoir reçu un accusé de connection de la part de la socket ou l'on se connect.

Donc envoyer un message sur une socket ou personnes n'est lié (bind) où sur une socket ROUTER ou aucune connection n'est active ne crée pas de file d'attente pour stocker les messages.

Quand dois je utiliser Bind et quand dois je utiliser Connect ?

En rêgle générale : utilisez "bind" sur vos noeuds  les plus stables de votre architecture et "connect" sur les noeuds les plus volatiles. Pour la pattern requete/réponse, le fournisseur de service doit être le noeud "bind" (serveur) et le client utilise "connect", comme une bonne vieille architecture client-serveur TCP.

Si vous ne pouvez pas définir de noeuds stables (ex. peer-to-peer), peut être faut-il un noeud stable suplémentaire au milieu, ou les autres noeuds se connectent.

Vous pouvez en lire plus à ce sujet sur la FAQ ZeroMQ http://zeromq.org/area:faq sous la section "Why do I see different behavior when I bind a socket versus connect a socket?".



## Multipart messages

ZeroMQ/NetMQ utilise le concepte de Frame. Les messages sont composés de une ou plusieurs Frames. NetMQ fournit des méthodes pour envoyer des String mais vous devriez vous familiariser avec le concept de multiFrame et voir comment cela fonctionne.

Vous pouvez avoir plus de détails dans la partie [Message](http://netmq.readthedocs.org/fr/latest/message/)


## Patterns

<a href="http://zguide.zeromq.org/page:all" target="_blank">ZeroMQ</a> (et bien sur NetMQ) n'utilise que des conceptes de pattern et de bloc de construction. Le <a href="http://zguide.zeromq.org/page:all" target="_blank">guide ZeroMQ</a> définit tout ce que vous devez savoir à propos de ces patterns. Lisez les sections suivantes du guide avant de commencer à travailler avec NetMQ.

+ <a href="http://zguide.zeromq.org/page:all#Chapter-Sockets-and-Patterns" target="_blank">Chapter 2 - Sockets and Patterns</a>
+ <a href="http://zguide.zeromq.org/page:all#Chapter-Advanced-Request-Reply-Patterns" target="_blank">Chapter 3 - Advanced Request-Reply Patterns</a>
+ <a href="http://zguide.zeromq.org/page:all#Chapter-Reliable-Request-Reply-Patterns" target="_blank">Chapter 4 - Reliable Request-Reply Patterns</a>
+ <a href="http://zguide.zeromq.org/page:all#Chapter-Advanced-Pub-Sub-Patterns" target="_blank">Chapter 5 - Advanced Pub-Sub Patterns</a>


NetMQ a aussi des exemples de quelques-une de ces patterns écrites avec l'Api NetMQ. Une fois votre pattern trouvée dans le  <a href="http://zguide.zeromq.org/page:all" target="_blank">guide ZeroMQ</a> c'est simple de la reproduire avec l'Api NetMQ.

Voici quelques liens montrant des patterns reproduites avec l'api NetMQ :

+ <a href="https://github.com/zeromq/netmq/tree/master/src/Samples/Brokerless%20Reliability%20(Freelance%20Pattern)/Model%20One" target="_blank">Brokerless Reliability Pattern - Freelance Model one</a>
+ <a href="https://github.com/zeromq/netmq/tree/master/src/Samples/Load%20Balancing%20Pattern" target="_blank">Load Balancer Patterns</a>
+ <a href="https://github.com/zeromq/netmq/tree/master/src/Samples/Pirate%20Pattern/Lazy%20Pirat" target="_blank">Lazy Pirate Pattern</a>
+ <a href="https://github.com/zeromq/netmq/tree/master/src/Samples/Pirate%20Pattern/Simple%20Pirate" target="_blank">Simple Pirate Pattern</a>


Pour les autres patterns, le <a href="http://zguide.zeromq.org/page:all" target="_blank">guide ZeroMQ</a> doit être votre premier réflexe.

Les patterns ZeroMQ sont implémentées avec des paires de sockets de différents types. En d'autre termes, pour comprendre les patterns ZeroMQ, vous devez comprendre les différents types de socket et comment ils fonctionnent ensembles.


Les patterns de base de ZeroMQ sont :


+ **Request-reply**, qui connectent un ensemble de clients à un ensemble de services. Utilisable pour faire de l'appel de méthode à distance ou la mise en place de patterns de distribution de taches.
+ **Pub-sub**, qui connectent un ensemble de fournisseurs à un ensemble d'abonnés. C'est une pattern de distribution de donnée.
+ **Pipeline**, qui sert à envoyer des données à différents noeuds organisés en mode pipeline. Cela sert pour des patterns d'éxecution de taches en parallèles et des patterns de collection de données.
+ **Exclusive pair**, qui connectent deux sockets exclusivement. Cela sert à connecter deux thread dans un process. A ne pas confondre avec les paires de socket dites "normales".

Ceci est une liste des combinaisons (connect->bind) possibles entre les différents types de sockets (bien que bind soit utilisable des deux cotés):

+ <code>PublisherSocket</code> et <code>SubscriberSocket</code>
+ <code>RequestSocket</code> et <code>ResponseSocket</code>
+ <code>RequestSocket</code>  et <code>RouterSocket</code>
+ <code>DealerSocket</code> et <code>ResponseSocket</code>
+ <code>DealerSocket</code> et <code>RouterSocket</code>
+ <code>DealerSocket</code> et <code>DealerSocket</code>
+ <code>RouterSocket</code> et <code>RouterSocket</code>
+ <code>PushSocket</code> et <code>PullSocket</code>
+ <code>PairSocket</code> et <code>PairSocket</code>

Toutes les autres combinaisons vont générés des résultats inattendus et zeroMQ renverra surement une erreurs dans ce cas.


## Options

NetMQ possède plusieurs options qui affectent le déroulement des choses.

Vous allez surement, suivant l'architecture que vous voulez créer ou les types de sockets utilisés, jouer avec ces options.
Dans NetMQ, elles sont définies dans la propriété xxxxSocket.Options.

Il est difficile de dire exactement quelles options vous devez mettre étant donné que celà dépend grandement de ce que vous essayez de faire.Voici une liste de ces options :

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

Nous ne détaillerons pas ici ces options mais rentrerons dans le détail là ou elle sont utilisés.
D'une manière générale, si dans le <a href="http://zguide.zeromq.org/page:all" target="_blank">guide ZeroMQ</a> une option est mentionnée, c est quelle doit être utilisée dans la pattern.


