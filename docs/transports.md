Protocoles de Transport
=====

NetMQ supporte 3 protocoles

+ TCP
+ InProc
+ PGM




## TCP

TCP est le protocole le plus commun. La plupart des exemple utilise ce protocole. Voici un exemple d'utilisation.



    using System;
    using System.Collections.Generic;
    using System.Linq;
    using System.Text;
    using System.Threading;
    using System.Threading.Tasks;
    using NetMQ;

    namespace Tcp
    {
        class Program
        {
            static void Main(string[] args)
            {
                using (var context = NetMQContext.Create())
                {
                    using (var server = context.CreateResponseSocket())
                    {
                        server.Bind("tcp://*:5555");

                        using (var client = context.CreateRequestSocket())
                        {
                            client.Connect("tcp://localhost:5555");
                            Console.WriteLine("Sending Hello");
                            client.Send("Hello");

                            var message = server.ReceiveString();
                            Console.WriteLine("Received {0}", message);

                            Console.WriteLine("Sending World");
                            server.Send("World");

                            message = client.ReceiveString();
                            Console.WriteLine("Received {0}", message);

                            Console.ReadLine();
                        }

                    }
                }
            }
        }
    }

Qui après lancement donne ce résultat :

<p>
<i>
Sending Hello<br/>
Received Hello<br/>
Sending World<br/>
Received World<br/>
</i>
</p>

Nous pouvons voir que le format pour les informations TCP spécifiées dans les méthodes <code>Bind()</code> et <code>Connect()</code> sont de cette forme :

tcp://*:5555

Il y a 3 parties : 

+ [0] la définition du protocole utilisé "tcp".
+ [1] la définition de l'adresse ip utilisé ("*" veut dire n'importe quelle adresse)
+ [2] le numéro de port de connection (ici "5555")




## InProc


InProc (In process) vous permet de vous connecter dans différentes parties du process actuel.C'est très utile pour les cas suivant :
 
+ Communication Inter-Thread 
+ Partage de données entre Thread -> Ne plus s'encombrer des locks. Quand vous envoyez une donnée dans des sockets, chaque paire de socket à une copie des données.
+ Communiquer entre différentes partie du system. (par exemple à la place d'avoir une class Statique)

NetMQ possède plusieurs composants utilisant InProc, comme les modèles [Actor] (https://github.com/zeromq/netmq/blob/master/docs/actor.md) et [Devices] (https://github.com/zeromq/netmq/blob/master/docs/devices.md).

Pour l'instant, afin de montrer une simple utilisation de Inproc, essayons d'envoyer une String (pour rester simplke mais ceci pourrait être un objet sérializé) entre 2 thread.

Un peu de code :


    using System;
    using System.Collections.Generic;
    using System.Linq;
    using System.Text;
    using System.Threading;
    using System.Threading.Tasks;
    using NetMQ;

    namespace Client
    {
        class Program
        {
            public async Task Start()
            {
                using (var context = NetMQContext.Create())
                {
                    var end1 = context.CreatePairSocket();
                    var end2 = context.CreatePairSocket();
                    end1.Bind("inproc://InprocTest_5555");
                    end2.Connect("inproc://InprocTest_5555");
                    var end1Task = Task.Run(() =>
                    {
                        Console.WriteLine("ManagedThreadId = {0}", Thread.CurrentThread.ManagedThreadId);
                        Console.WriteLine("Sending hello down the inproc pipeline");
                        end1.Send("Hello");
                    
                    });
                    var end2Task = Task.Run(() =>
                    {
                        Console.WriteLine("ManagedThreadId = {0}", Thread.CurrentThread.ManagedThreadId);
                        var message = end2.ReceiveString();
                        Console.WriteLine(message);
                    
                    });
                    Task.WaitAll(new[] {end1Task, end2Task}); 
                    end1.Dispose();
                    end2.Dispose();
                }
            }


            static void Main(string[] args)
            {
                Program p = new Program();
                p.Start().Wait();
                Console.ReadLine();
            }
        }
    }


Qui après lancement donne ce résultat :

<p>
<i>
ManagedThreadId = 12<br/>
ManagedThreadId = 6<br/>
Sending hello down the inproc pipeline<br/>
Hello<br/>
</i>
</p>



Nous pouvons voir que le format pour les informations InProc spécifiées dans les méthodes <code>Bind()</code> et <code>Connect()</code> sont de cette forme :

**inproc://InprocTest_5555**

Il y a 2 parties : 

+ [0] la définition du protocole utilisé "inproc".
+ [1] une string unique définissant le canal ("InprocTest_5555")



## PGM

"Pragmatic General Multicast" (PGM) est un protocol de transport fiable pour les applications nécessitant la publication de donnée de manière ordonné ou desordonné à de multiple clients.
 
Pgm guaranti qu'un client dans un groupe va recevoir toutes les données émises ou sera capables de détecter si des données ont été perdues. Le but principale de Pgm est la simplicité des opérations tout en permettant une forte scabilité et l'efficacité des échanges.

Pour utiliser PGM avec NetMQ,nous devons seulement suivre ces trois rêgles :

1. Les types de socket utilisés sont <code>PublisherSocket</code> et <code>SubscriberSocket</code>
   qui sont detaillés dans la partie [pub-sub] (https://github.com/zeromq/netmq/blob/master/docs/pub-sub.md) de la documentation.
2. Etre sûr de lancer l'application en mode "Administrateur"
3. Etre sûr d'avoir activer le "Support Multicast". Vous pouvez le faire comme celà :


<br/>
<img src="https://raw.githubusercontent.com/zeromq/netmq/master/docs/Images/PgmSettingsInWindows.png"/>



Voici une petite démo utilisant PGM, ainsi que <code>PublisherSocket</code> et <code>SubscriberSocket</code> et quelques options.



    using System;
    using NetMQ;

    namespace Pgm
    {
        class Program
        {
            static void Main(string[] args)
            {
                const int MegaBit = 1024;
                const int MegaByte = 1024;
                using (NetMQContext context = NetMQContext.Create())
                {
                    using (var pub = context.CreatePublisherSocket())
                    {
                        pub.Options.MulticastHops = 2;
                        pub.Options.MulticastRate = 40 * MegaBit; // 40 megabit
                        pub.Options.MulticastRecoveryInterval = TimeSpan.FromMinutes(10);
                        pub.Options.SendBuffer = MegaByte * 10; // 10 megabyte
                        pub.Connect("pgm://224.0.0.1:5555");
                        using (var sub = context.CreateSubscriberSocket())
                        {
                            using (var sub2 = context.CreateSubscriberSocket())
                            {
                                sub.Options.ReceivevBuffer = MegaByte * 10;
                                sub2.Options.ReceivevBuffer = MegaByte * 10;
                                sub.Bind("pgm://224.0.0.1:5555");
                                sub2.Bind("pgm://224.0.0.1:5555");
                                sub.Subscribe("");
                                sub2.Subscribe("");
                                Console.WriteLine("Server sending 'Hi'");
                                pub.Send("Hi");
                                bool more;
                                string message = sub.ReceiveString(out more);
                                Console.WriteLine("sub message = '{0}'", message);
                                message = sub2.ReceiveString(out more);
                                Console.WriteLine("sub2 message = '{0}'", message);


                                Console.ReadLine();
                            }
                        }
                    }
                }
            }
        }
    }


Qui après lancement donne ce résultat :

<p>
<i>
Server sending 'Hi'<br/>
sub message = 'Hi'<br/>
sub2 message = 'Hi'<br/>
</i>
</p>



Nous pouvons voir que le format pour les informations PGM spécifiées dans les méthodes <code>Bind()</code> et <code>Connect()</code> sont de cette forme :

pgm://224.0.0.1:5555

Il y a 3 parties :

+ [0] la définition du protocole utilisé "pgm".
+ [1] la définition de l'adresse ip utilisé ("224.0.0.1"). "*" est possible.
+ [2] le numéro de port de connection (ici "5555")


Une autre bonne ressource pour avoir des information sur Pgm est le [PgmTests] (https://github.com/zeromq/netmq/blob/master/src/NetMQ.Tests/PgmTests.cs)



