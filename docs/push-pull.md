Push / Pull
=====

NetMq implémente les <code>PushSocket</code> et <code>PullSocket</code>. Voyons coimment les utiliser

Normalement ces deux socket vont de pair. Une <code>PushSocket</code> va envoyer des données à une <code>PullSocket</code>, tandis qu'une <code>PullSocket</code> s'attend a recevoir des données d'une ou plusieurs <code>PushSocket</code>. Jusqu'ici pas de problèmes!

Vous pouvez utiliser cette configuration pour faire une architecture permettant par exemple de distribués du travail, un peu comme la patterne <a href="http://zguide.zeromq.org/page:all#Divide-and-Conquer" target="_blank">divide and conquer</a>.

L'idée est d'avoir une partie qui génère du travail et qui le distribue à 1-n "travailleurs". Chaque "travailleur" fait son boulot et renvoi ses résultat au process (peut aussi etre un thread) ou les résultats sont traités.

Dans le <a href="http://zguide.zeromq.org/page:all" target="_blank">guide ZeroMQ</a>, il y a un exemple qui montre comment le ditribueur de tache dit aux travailleurs d'attendre pendant une certaine periode.

Nous allons essayés de créer un exemple un peu plus élaboré, le distributeur de tache va envoyer le temps que chaque travailleur doit attendre (simulation d'un travail a faire).

En réalité , ce travail pourrait être n'importe quelle tache, du moment que ce sont des taches qui peuvent se découper en sous-tâches bien séparées, peut importe le nombre de "travailleurs".

Voici un diagramme de ce que nous essayons de faire :

<br/>
<br/>
<img src="https://raw.githubusercontent.com/zeromq/netmq/master/docs/Images/Fanout.png"/>


<br/>
<br/>

et le code....

**Ventilator**


    using System;
    using NetMQ;


    namespace Ventilator
    {
        public class Program
        {
            public static void Main(string[] args)
            {
                // Task Ventilator
                // Binds PUSH socket to tcp://localhost:5557
                // Sends batch of tasks to workers via that socket
                Console.WriteLine("====== VENTILATOR ======");

                using (NetMQContext ctx = NetMQContext.Create())
                {
                    //socket to send messages on
                    using (var sender = ctx.CreatePushSocket())
                    {
                        sender.Bind("tcp://*:5557");

                        using (var sink = ctx.CreatePushSocket())
                        {
                            sink.Connect("tcp://localhost:5558");

                            Console.WriteLine("Press enter when worker are ready");
                            Console.ReadLine();

                            //the first message it "0" and signals start of batch
                            //see the Sink.csproj Program.cs file for where this is used
                            Console.WriteLine("Sending start of batch to Sink");
                            sink.Send("0");



                            Console.WriteLine("Sending tasks to workers");

                            //initialise random number generator
                            Random rand = new Random(0);

                            //expected costs in Ms
                            int totalMs = 0;

                            //send 100 tasks (workload for tasks, is just some random sleep time that
                            //the workers can perform, in real life each work would do more than sleep
                            for (int taskNumber = 0; taskNumber < 100; taskNumber++)
                            {
                                //Random workload from 1 to 100 msec
                                int workload = rand.Next(0, 100);
                                totalMs += workload;
                                Console.WriteLine("Workload : {0}", workload);
                                sender.Send(workload.ToString());
                            }
                            Console.WriteLine("Total expected cost : {0} msec", totalMs);
                            Console.WriteLine("Press Enter to quit");
                            Console.ReadLine();
                        }
                    }
                }
            }
        }
    }


**Worker**


    using System;
    using System.Threading;
    using NetMQ;


    namespace Worker
    {
        public class Program
        {
            public static void Main(string[] args)
            {
                // Task Worker
                // Connects PULL socket to tcp://localhost:5557
                // collects workload for socket from Ventilator via that socket
                // Connects PUSH socket to tcp://localhost:5558
                // Sends results to Sink via that socket
                Console.WriteLine("====== WORKER ======");


                using (NetMQContext ctx = NetMQContext.Create())
                {
                    //socket to receive messages on
                    using (var receiver = ctx.CreatePullSocket())
                    {
                        receiver.Connect("tcp://localhost:5557");

                        //socket to send messages on
                        using (var sender = ctx.CreatePushSocket())
                        {
                            sender.Connect("tcp://localhost:5558");

                            //process tasks forever
                            while (true)
                            {
                                //workload from the vetilator is a simple delay
                                //to simulate some work being done, see
                                //Ventilator.csproj Proram.cs for the workload sent
                                //In real life some more meaningful work would be done
                                string workload = receiver.ReceiveString();

                                //simulate some work being done
                                Thread.Sleep(int.Parse(workload));

                                //send results to sink, sink just needs to know worker
                                //is done, message content is not important, just the precence of
                                //a message means worker is done. 
                                //See Sink.csproj Proram.cs 
                                Console.WriteLine("Sending to Sink");
                                sender.Send(string.Empty);
                            }
                        }

                    }
                }

            }
        }
    }


**Sink**


    using System;
    using System.Diagnostics;
    using System.Threading;
    using System.Threading.Tasks;
    using NetMQ;


    namespace Sink
    {
        public class Program
        {
            public static void Main(string[] args)
            {

                // Task Sink
                // Bindd PULL socket to tcp://localhost:5558
                // Collects results from workers via that socket
                Console.WriteLine("====== SINK ======");

                using (NetMQContext ctx = NetMQContext.Create())
                {
                    //socket to receive messages on
                    using (var receiver = ctx.CreatePullSocket())
                    {
                        receiver.Bind("tcp://localhost:5558");

                        //wait for start of batch (see Ventilator.csproj Program.cs)
                        var startOfBatchTrigger = receiver.ReceiveString();
                        Console.WriteLine("Seen start of batch");

                        //Start our clock now
                        Stopwatch watch = new Stopwatch();
                        watch.Start();

                        for (int taskNumber = 0; taskNumber < 100; taskNumber++)
                        {
                            var workerDoneTrigger = receiver.ReceiveString();
                            if (taskNumber % 10 == 0)
                            {
                                Console.Write(":");
                            }
                            else
                            {
                                Console.Write(".");
                            }
                        }
                        watch.Stop();
                        //Calculate and report duration of batch
                        Console.WriteLine();
                        Console.WriteLine("Total elapsed time {0} msec", watch.ElapsedMilliseconds);
                        Console.ReadLine();
                    }
                }
            }
        }
    }




Pour lancer le programme, ces 3 fichiers Bat vous serons utiles.


<br/>
<br/>
**Run1Worker.bat**
<br/>
<br/>
cd Ventilator/bin/Debug<br/>
start Ventilator.exe<br/>
cd../../..<br/>
cd Sink/bin/Debug<br/>
start Sink.exe<br/>
cd../../..<br/> 
cd Worker/bin/Debug<br/>
start Worker.exe<br/>


Vous devrier obtenir en résultat pour 1 travailleur :

<i>
====== SINK ======<br/>
Seen start of batch<br/>
:………:………:………:………:………:………:………:………<br/>
:………:………<br/>
Total elapsed time 5695 msec<br/>
</i>




<br/>
<br/>
**Run2Workers.bat**
<br/>
<br/>
cd Ventilator/bin/Debug<br/>
start Ventilator.exe<br/>
cd../../..<br/>
cd Sink/bin/Debug<br/>
start Sink.exe<br/>
cd../../..<br/>
cd Worker/bin/Debug<br/>
start Worker.exe<br/>
start Worker.exe<br/>



Vous devrier obtenir en résultat pour 2 travailleurs :

<i>
====== SINK ======<br/>
Seen start of batch<br/>
:………:………:………:………:………:………:………:………<br/>
:………:………<br/>
Total elapsed time 2959 msec<br/>
</i>


<br/>
<br/>
**Run4Workers.bat**
<br/>
<br/>
cd Ventilator/bin/Debug<br/>
start Ventilator.exe<br/>
cd../../..<br/>
cd Sink/bin/Debug<br/>
start Sink.exe<br/>
cd../../..<br/> 
cd Worker/bin/Debug<br/>
start Worker.exe<br/>
start Worker.exe<br/>
start Worker.exe<br/>
start Worker.exe<br/>


Vous devrier obtenir en résultat pour 4 travailleurs :


<i>
====== SINK ======<br/>
Seen start of batch<br/>
:………:………:………:………:………:………:………:………<br/>
:………:………<br/>
Total elapsed time 1492 msec<br/>
</i>


<br/>
<br/>

On voit bien ici que plus on augmente le nombre de travailleurs, plus le temps d'execution des taches diminue.

Il y a cependant quelques points d'attentions sur cette patterne.

+ Le <code>Ventilator</code> utilise une NetMQ <code>PushSocket</code> pour distribuer le travail aux <code>Worker</code>s, c'est du load balencing
+ Le <code>Ventilator</code> et le <code>Sink</code> sont les parties statique de l'architecture, alors que les <code>Worker</code>s sont dynamiques.
+ Nous devons synchroniser le debut du batch (Quand les <code>Worker</code>s sont prêts), sinon le premier <code>Worker</code> quyi se connectera aura plus de messages que les autres, ce qui n'est pas vraiment du load balencing dans ce cas.
+ Le <code>Sink</code> utilise une NetMQ <code>PullSocket</code> pour traiter les résultats des <code>Worker</code>s
