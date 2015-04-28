XPub / XSub
=====


<i>
L'un des problèmes lorsque l'on design une architecture ditribuées qui peut s'agrandir est la découverte des autres. Comment chaques noeuds connait la présence des autres. C'est spécialement difficile lorsque les noeuds sont dynamique. Nous appelons cela le "probleme de découverte dynamique";
<br/>
<br/>
Il y a plusieurs solution à ce problème. La plus simple est de simplement l'éviter en "Hard-Codant" (configuration) l'architecture du réseau, la découverte est donc faite à la main. Quand vous avez un nouveau noeud, vous le déclarez dans un fichier de configuration.
<br/>
<br/>
<img src="https://github.com/imatix/zguide/raw/master/images/fig12.png"/>
<br/>
<br/>
En principe, c'est une architecture extremement fragile. Imaginez que vous ayez une centaine de clients et un server. Vous configurez chaques clients pour qu'ils connaissent l'adresse du serveur. Si vous ajouter un client, aucun problème. Ca se complique lorsque vous décidez d'ajouter un serveur...
<br/>
<br/>
<img src="https://github.com/imatix/zguide/raw/master/images/fig13.png"/>
<br/>
<br/>
L'autre manière est d'avoir un intermédiaire. Cet intermédiaire est statique. Dans une architecture de messaging, c'est ce que l'on appelle le "broker". ZeroMQ ne contient pas de "broker" en tant que tel, mais il permet d'en définir un assez facilement.
<br/>
<br/>
Vous devez vous demander, si tous les réseaux deviennent assez grand pour avoir besoin d'intermédiaire, pourquoi ne mettons nous pas simplement un message broker en place dans chaque application ? Pour les débutants, une topology en étoile est un bon compromis. Mais un message broker est une partie difficile et complexe et peut devenir rapidement un probleme.
<br/>
<br/>
Il est plus facile de penser l'intermédiaire comme un simple diffuseur de message sans état. Un ebonne analogie est le proxy Http. Ca n'a pas de fonction spéciale mis à part diffusée la requete. Ajouter un PUB-SUB résoud notre problème de découverte dynamique dans notre exemple. Nous définissons le "proxy" comme étant au centre de notre réseau. Le proxy ouvre une XSUB Socket, une XPUB Socket et bind chaqu une d'elle a une adresse statique du réseau. Ensuite chaqu'un se connecte au proxy à la place de se connecter directement aux autres noeuds. Cela devient facile d'ajouter des subscriber et des publisher.
<br/>
<br/>
Nous avons besoin de XPUB et XSUB car ZeroMQ fait un suivi des abonnements entre les publisher et les subscriber. XSUB et XPUB sont comme SUB et PUB, à l'exception qu'elles exposent ces abonnements comme étant des messages spéciaux. Le proxy doit faire suivre ces message spéciaux d'abonnements du subscriber vers le publisher, en les lisant dans la socket XSUB et en les écrivant dans la socket XPUB. C'est la principale utilité de XSUB et XPUB.
</i>
<br/>
<br/>
Le texte ci dessus est tiré de <a href="ZeroMQ guide - dynamix discovery problem" target="_blank">http://zguide.zeromq.org/page:all#The-Dynamic-Discovery-Problem</a>.


## Un exemple

Maintenant que vous savez pourquoi utiliser XPUB et XSUB, regardons un exemple. Cet exemple est très proche de l'exemple du guide ZeroMQ, <a href="ZeroMQ guide - dynamix discovery problem" target="_blank">http://zguide.zeromq.org/page:all#The-Dynamic-Discovery-Problem</a>, et est séparé en trois composants :

+ Publisher
+ Intermediary
+ Subscriber

Voiuci le code des différents composants:

**Publisher**

La <code>PublisherSocket</code> se connecte à la <code>XSubscriberSocket</code> adresse.

    using System;
    using NetMQ;

    namespace XPubXSub.Publisher
    {
        class Program
        {
            static void Main(string[] args)
            {

                Random rand = new Random(50);

                const string xsubAddress = "tcp://127.0.0.1:5678"; 


                using (var context = NetMQContext.Create())
                {
                    using (var pubSocket = context.CreatePublisherSocket())
                    {
                        Console.WriteLine("Publisher socket binding...");
                        pubSocket.Options.SendHighWatermark = 1000;
                        pubSocket.Connect(xsubAddress);



                        while(true)
                        {
                            var randomizedTopic = rand.NextDouble();
                            if (randomizedTopic > 0.5)
                            {
                                var msg = "TopicA msg-" + randomizedTopic;
                                Console.WriteLine("Sending message : {0}", msg);
                                pubSocket.SendMore("TopicA").Send(msg);
                            }
                            else
                            {
                                var msg = "TopicB msg-" + randomizedTopic;
                                Console.WriteLine("Sending message : {0}", msg);
                                pubSocket.SendMore("TopicB").Send(msg);
                            }
                        }
                    }
                }

            }
        }
    }



**Intermediary**

L'intermédiaire (Le processus qui contient les socket <code>XPublisherSocket</code> / <code>XSubscriberSocket</code>) sont responsable du relais du message comme définis:

+ De la <code>XPublisherSocket</code> à la <code>XSubscriberSocket</code>
+ De la <code>XSubscriberSocket</code> à la <code>XPublisherSocket</code>

Nous utilisons le <code>Poller</code> NeTMQ, mais il y a une meilleur manière en utilisant le <code>Proxy</code> NeTMQ, qui vous permet de définir de socket que vous voulez "proxifiez". Le <code>Proxy</code> NetMQ se charge d'envoyer les messages d'une socket à l'autre et vice versa.

Voici le code :

    using System;
    using System.Threading.Tasks;
    using NetMQ;

    namespace XPubXSub.Intermediary
    {
        class Program
        {
            static void Main(string[] args)
            {

                const string xpubAddress = "tcp://127.0.0.1:1234";
                const string xsubAddress = "tcp://127.0.0.1:5678"; 

                using (var context = NetMQContext.Create())
                {
                    using (var xpubSocket = context.CreateXPublisherSocket())
                    {
                        xpubSocket.Bind(xpubAddress);

                        using (var xsubSocket = context.CreateXSubscriberSocket())
                        {

                            xsubSocket.Bind(xsubAddress);

                            Console.WriteLine("Intermediary started, and waiting for messages");


                            //Use the Proxy class to proxy between frontend / backend
                            NetMQ.Proxy proxy = new Proxy(xsubSocket, xpubSocket,null);
                            Task.Factory.StartNew(proxy.Start);

                            Console.ReadLine();
                        }

                    }
                }
            }
        }
    }
 


**Subscriber**

Nous pouvons voir que la socket <code>SubscriberSocket</code> se connecte à la <code>XPublisherSocket</code> address

    using System;
    using System.Collections.Generic;
    using NetMQ;

    namespace XPubXSub.Subscriber
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

                const string xpubPortAddress = "tcp://127.0.0.1:1234";   
 
                using (var context = NetMQContext.Create())
                {
                    using (var subSocket = context.CreateSubscriberSocket())
                    {
                        subSocket.Options.ReceiveHighWatermark = 1000;
                        subSocket.Connect(string.Format(xpubPortAddress));
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

Vous devriez obtenir ceci comme résultat (cela dépend du nombre de subscriber , et à quels topics ils sont inscrits. Mais il n'est pas difficile d'en ajouter ou d'ajouter des publishers)
<br/>
<br/>
<img src="https://raw.githubusercontent.com/zeromq/netmq/master/docs/Images/XPubXSubDemo.png"/>
<br/>
<br/>



Voici 4 fichiers BAT qui vous aideront...


**RunXPubXSub.bat**
<br/>
start RunXPubXSubIntermediary.bat<br/>
start RunXPublisher.bat<br/>
<br/>
<br/>
start RunXSubscriber "TopicA"<br/>
start RunXSubscriber "TopicB"<br/>
<br/>
<br/>

**RunXPublisher.bat**
<br/>
cd XPubXSub.Publisher\bin\Debug<br/>
XPubXSub.Publisher.exe<br/>
<br/>
<br/>

**RunXPubXSubIntermediary.bat**
<br/>
cd XPubXSub.Intermediary\bin\Debug<br/>
XPubXSub.Intermediary.exe<br/>
<br/>
<br/>

**RunXSubscriber.bat**
<br/>
set "topic=%~1"<br/>
<br/>
<br/>
cd XPubXSub.Subscriber\bin\Debug<br/>
XPubXSub.Subscriber.exe %topic%<br/>
