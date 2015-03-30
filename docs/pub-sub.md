Pub/Pub
=====

NetMQ gère la pattern Pub/Sub avec ces deux socket :

+ <code>PublisherSocket</code>
+ <code>SubscriberSocket</code>

Comme d'habitude, on les crées à partir du <code>NetMQContext</code> avec la méthode <code>.CreateXXXSocket()</code>.
Ce qui donne : 

+ <code>CreatePublisherSocket()</code>
+ <code>CreateSubscriberSocket()</code>


## Topics

NetMQ permet l'utilisation de topics (canaux), <code>PublisherSocket</code> envoie la "frame" 1( [messages documentation page](https://github.com/zeromq/netmq/blob/master/docs/message.md)) du message qui contiendra le topic et dans la frame 2 le message en lui même, ce qui donne :

<table CellSpacing="0" Padding="0">
<tr bgcolor="LightGray">
<th width="100px" style="text-align:center; ">Frame 1</th>
<th width="400px" style="text-align:center; ">Frame 2</th>
</tr>
<tr>
<td width="80px" style="text-align:center; ">TopicA</td>
<td width="400px" style="text-align:center;">C'est un message dans le 'TopicA'</td>
</tr>
</table>

<br/>
<br/>
Voici comment envoyer les deux frames (Vous pouvez aussi utiliser un <code>NetMQMessage</code> et ajouter les frames une par une au message):
<br/>
<br/>


    pubSocket.SendMore("TopicA").Send("C'est un message dans le 'TopicA'");

<br/>
<br/>
La <code>SubscriberSocket</code> peut s'abonner à un certain topic pour recevoir les messages, ce qui est faisable en passant le nom du topic dans la méthode <code>Subscribe()</code> de <code>SubscriberSocket</code>.
<br/>
<br/>

Voici un exemple:
<br/>
<br/>

    subSocket.Subscribe("TopicA");

<br/>
<br/>
## Comment s'abonner à tous les messages ?

Il est possible de s'abonner à tous les messages en mettant une chaine de caractère vide à la méthode <code>subscriberSocket.Subscribe()</code>. 


## Exemple

Cet exemple est très simple et suis ces rêgles : 

+ Il y a un publisher qui crée, soit des messages pour le  topicA', soit des messages pour le 'topicB' (dépend d'un nombre aléatoire)
+ Il y a un Subscriber générique (le nom du topic auquel il s'abonne est passé en paramètre dans la ligne de commande)

Voici le code:

**Publisher**

    using System;
    using System.Threading;
    using NetMQ;

    namespace Publisher
    {
        class Program
        {
            static void Main(string[] args)
            {
                Random rand = new Random(50);

                using (var context = NetMQContext.Create())
                {
                    using (var pubSocket = context.CreatePublisherSocket())
                    {
                        Console.WriteLine("Publisher socket binding...");
                        pubSocket.Options.SendHighWatermark = 1000;
                        pubSocket.Bind("tcp://localhost:12345");

                        for (var i = 0; i < 100; i++)
                        {
                            var randomizedTopic = rand.NextDouble();
                            if (randomizedTopic > 0.5)
                            {
                                var msg = "TopicA msg-" + i;
                                Console.WriteLine("Sending message : {0}", msg);
                                pubSocket.SendMore("TopicA").Send(msg);
                            }
                            else
                            {
                                var msg = "TopicB msg-" + i;
                                Console.WriteLine("Sending message : {0}", msg);
                                pubSocket.SendMore("TopicB").Send(msg);
                            }

                            Thread.Sleep(500);
                        }
                    }
                }
            }
        }
    }


**Subscriber**

    using System;
    using System.Collections.Generic;
    using System.Diagnostics;
    using System.Linq;
    using System.Text;
    using System.Threading.Tasks;
    using NetMQ;

    namespace SubscriberA
    {
        class Program
        {

            public static List<string> allowableCommandLineArgs = new List<string>();

            static Program()
            {
                allowableCommandLineArgs.Add("TopicA");
                allowableCommandLineArgs.Add("TopicB");
                allowableCommandLineArgs.Add("All");
            }

            static void PrintUsageAndExit()
            {
                Console.WriteLine("Subscriber is expected to be started with either 'TopicA', 'TopicB' or 'All'");
                Console.ReadLine();
                Environment.Exit(-1);
            }

            static void Main(string[] args)
            {

                if (args.Length != 1)
                {
                    PrintUsageAndExit();
                }

                if (!allowableCommandLineArgs.Contains(args[0]))
                {
                    PrintUsageAndExit();
                }

                string topic = args[0] == "All" ? "" : args[0];
                Console.WriteLine("Subscriber started for Topic : {0}", topic);

                using (var context = NetMQContext.Create())
                {
                    using (var subSocket = context.CreateSubscriberSocket())
                    {
                        subSocket.Options.ReceiveHighWatermark = 1000;
                        subSocket.Connect("tcp://localhost:12345");
                        subSocket.Subscribe(topic);
                        Console.WriteLine("Subscriber socket connecting...");
                        while (true)
                        {
                            string messageTopicReceived = subSocket.ReceiveString();
                            string messageReceived = subSocket.ReceiveString();
                            Console.WriteLine(messageReceived);
                        }
                    }
                }            
            }
        }
    }


Pour lancer le programme, ces 3 fichiers BAT seront utiles.


**RunPubSub.bat**
<br/>
start RunPublisher.bat<br/>
<br/>
<br/>
start RunSubscriber "TopicA"<br/>
start RunSubscriber "TopicB"<br/>
start RunSubscriber "All"<br/>


**RunPublisher.bat**
<br/>
<br/>
cd Publisher\bin\Debug<br/>
Publisher.exe<br/>

**RunSubscriber.bat**
<br/>
<br/>
set "topic=%~1"<br/>
cd Subscriber\bin\Debug<br/>
Subscriber.exe %topic%<br/>



Vous devriez voir ceci :


<br/>
<br/>
<img src="https://raw.githubusercontent.com/zeromq/netmq/master/docs/Images/PubSubUsingTopics.png"/>




Autres Considérations
=====

**HighWaterMark**


Les options <code>SendHighWaterMark/ReceiveHighWaterMark</code> spécifie le 'high water mark' pour la socket. Le 'high water mark' est une limite spécifiant le nombre maximum de message pouvant être empilés entre deux socket.

Si cette limite est atteinte et suivant le type de socket, la socket rentre dans un état spéciale et soit elle bloque les messages, soit elle supprime les messages suivants.

La valeur <code>SendHighWaterMark/ReceiveHighWaterMark</code> par defaut est 0 ce qui veux dire "sans limite".

Vous pouvez voir ces deux options dans <code>xxxxSocket.Options</code> :

+  <code>pubSocket.Options.SendHighWatermark = 1000;</code>
+  <code>pubSocket.Options.ReceiveHighWatermark = 1000;</code>


**Slow Subscribers**
<br/>
<br/>
Cette partie est couverte dans le <a href="http://zguide.zeromq.org/php:chapter5" target="_blank">guide ZeroMQ</a>


**Late Joining Subscribers**
<br/>
<br/>
Cette partie est couverte dans le <a href="http://zguide.zeromq.org/php:chapter5" target="_blank">guide ZeroMQ</a>
