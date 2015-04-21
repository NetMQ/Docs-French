Gérer plusieurs Sockets
=====

Pourquoi vouloir géréer plusieurs sockets en même temps ? Il existe plusieurs raison à cela : 

+ Vous avez plusieurs socket dans un même process qui sont reliées les unes aux autres et vous devez savoir quand chacunes d'elles est prête à recevoir des données.
+ Vous voulez avoir une requete ainsi qu'un publisher dans un même process.

Des fois vous allez avoir besoin de gérer plusieurs socket dans un même process. Et souvent vous voudrez utilisez les sockets que lorsqu'elles sont prêtes à être utilisées.

ZeroMq instaure ici le concept de <code>Poller</code> pour savoir si une socket est prête.

NetMQ a une implémentation du <code>Poller</code>, et il peut être utilisé pour :

+ Gérer une socket, pour savoir si la lecture est prête.
+ Gérer une liste de socket (<code>IEnumerable<NetMQSocket></code>) pour savoir si la lecture est prête.
+ Autoriser des <code>NetMQSocket</code>(s) a être ajouté dynamiquement et déterminer si les nouvelles requêtes sont prêtres à être lues.
+ Autorisé des <code>NetMQSocket</code>(s) à être supprimées dynamiquement.
+ Lever un evennement quan dune socket est prête.


## Les méthode du Poller 

Le <code>Poller</code> possède plusieurs méthodes qui vont vous aider à gerer tout ca. Plus précisément les méthodes <code>AddSocket(..)/RemoveSocket(..)</code> et <code>Start()/Stop()</code>.

Vous devez utiliser la méthode <code>AddSocket</code> pour ajouter dynamiquement une socket et vérifier son état de "prête pour la lecture", et ensuite appeller la méthode <code>Poller.Start()</code>.A ce moment, le <code>Poller</code> va appelle tous les evennements <code>ReceiveReady</code>.


## Poller Exemple

Maintenant que l'on sait à quoi sert le <code>Poller</code>, voyons un petit exemple. 

Dans le code suivant, une seule requête est ajouter au <code>Poller</code>. Nous pouvons voir que l'evennement <code>ReceiveReady</code> est gérer. Le <code>Poller</code> va automatiquement appeler l'evennement quand la requete sera "prête".


    using System;
    using System.Collections.Generic;
    using System.Linq;
    using System.Text;
    using System.Threading;
    using System.Threading.Tasks;

    using NetMQ;

    namespace ConsoleApplication1
    {
        class Program
        {
            static void Main(string[] args)
            {
                using (NetMQContext contex = NetMQContext.Create())
                {
                    using (var rep = contex.CreateResponseSocket())
                    {
                        rep.Bind("tcp://127.0.0.1:5002");

                        using (var req = contex.CreateRequestSocket())
                        using (Poller poller = new Poller())
                        {
                            req.Connect("tcp://127.0.0.1:5002");

                            //The ReceiveReady event is raised by the Poller
                            rep.ReceiveReady += (s, a) =>
                            {
                                bool more;
                                string messageIn = a.Socket.ReceiveString(out more);
                                Console.WriteLine("messageIn = {0}", messageIn);
                                a.Socket.Send("World");
                            };

                            poller.AddSocket(rep);

                            Task pollerTask = Task.Factory.StartNew(poller.Start);
                            req.Send("Hello");

                            bool more2;
                            string messageBack = req.ReceiveString(out more2);
                            Console.WriteLine("messageBack = {0}", messageBack);

                            poller.Stop();

                            Thread.Sleep(100);
                            pollerTask.Wait();
                        }
                    }
                }
                Console.ReadLine();
            }
        }
    }


Vous devriez voir dans la console :

<p>
<i>
messageIn = Hello<br/>  
messageBack = World<br/>
</i>
</p>



A partir de cet exemple, nous pouvons maintenant supprimer la <code>ResponseSocket</code> du <code>Poller</code> dès que nous voyons le premiers message, ce qui veux dire que nous ne devrions plus recevoir aucunes informations sur la <code>ResponseSocket</code>. 

Voici le nouveau code :


    using System;
    using System.Collections.Generic;
    using System.Linq;
    using System.Text;
    using System.Threading;
    using System.Threading.Tasks;

    using NetMQ;

    namespace ConsoleApplication1
    {
        class Program
        {
            static void Main(string[] args)
            {

                using (NetMQContext contex = NetMQContext.Create())
                {
                    using (var rep = contex.CreateResponseSocket())
                    {
                        rep.Bind("tcp://127.0.0.1:5002");

                        using (var req = contex.CreateRequestSocket())
                        using (Poller poller = new Poller())
                        {
                            req.Connect("tcp://127.0.0.1:5002");

                            //The ReceiveReady event is raised by the Poller
                            rep.ReceiveReady += (s, a) =>
                            {
                                bool more;
                                string messageIn = a.Socket.ReceiveString(out more);
                                Console.WriteLine("messageIn = {0}", messageIn);
                                a.Socket.Send("World");


                                //REMOVAL
                                //This time we remove the Socket from the Poller, so it should not receive any more messages
                                poller.RemoveSocket(a.Socket);
                            };

                            poller.AddSocket(rep);



                            Task pollerTask = Task.Factory.StartNew(poller.Start);
                            
                            req.Send("Hello");

                            bool more2;
                            string messageBack = req.ReceiveString(out more2);
                            Console.WriteLine("messageBack = {0}", messageBack);


        
                            //This should not do anything, as we removed the ResponseSocket
                            //the 1st time we sent a message to it
                            req.Send("Hello Again");

                           
                            Console.WriteLine("Carrying on doing the rest");

                            poller.Stop();

                            Thread.Sleep(100);
                            pollerTask.Wait();
                        }
                    }
                }
                Console.ReadLine();
            }
        }
    }


Ce qui donne :

<p><i>
messageIn = Hello<br/>  
messageBack = World<br/>
Carrying on doing the rest<br/>
</i>
</p>

Nous n'avons pas le message "Hello Again" que nous avons envoyés car la <code>ResponseSocket</code> a été supprimé du <code>Poller</code> avant.


## Timer(s)

Une autre fonctionnalité du <code>Poller</code> est de pouvoir gérer des <code>Timer</code> en appellant les méthodes <code>AddTimer(..) / RemoveTimer(..)</code>. 

Voici un exemple qui montre comment on ajoute un <code>NetMQTimer</code> qui attend 5 secondes. Le <code>NetMQTimer</code> est ajouter au <code>Poller</code>, qui va s'occuper de déclencher le <code>NetMQTimer.Elapsed</code>.

    using System;
    using System.Collections.Generic;
    using System.Linq;
    using System.Text;
    using System.Threading;
    using System.Threading.Tasks;

    using NetMQ;

    namespace ConsoleApplication1
    {
        class Program
        {
            static void Main(string[] args)
            {

                using (Poller poller = new Poller())
                {
                    NetMQTimer timer = new NetMQTimer(TimeSpan.FromSeconds(5));
                    timer.Elapsed += (s, a) =>
                            {
                                Console.WriteLine("Timer done");
                            }; ;
                    poller.AddTimer(timer);


                    Task pollerTask = Task.Factory.StartNew(poller.Start);


                    //give the poller enough time to run the timer (set at 5 seconds)
                    Thread.Sleep(10000);

                }

                Console.ReadLine();
            }
        }
    }


Ce qui donne :

<p><i>
Timer done<br/>  
</i>
</p>




## Pour aller plus loin

Pour plus d'utilisation du <code>Poller</code> vous pouvez aller sur <a href="https://github.com/zeromq/netmq/blob/master/src/NetMQ.Tests/PollerTests.cs" target="_blank">Poller tests</a>
