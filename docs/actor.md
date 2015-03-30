NetMQ Modèle Acteur
===

## Qu'est ce que le Modèle Acteur ?


Voici ce que dit wikipedia sur le modèle Acteur.

<p><i> En informatique, le modèle d'acteur est un modèle mathématique qui considère des acteurs comme les seules fonctions primitives nécessaires pour la programmation concurrente. Les acteurs communiquent par échange de messages. En réponse à un message, un acteur peut effectuer un traitement local, créer d'autres acteurs, ou envoyer d'autres messages.

….

….

Le modèle considère que tout est acteur. Un acteur est une entité capable de calculer, qui, en réponse à un message reçu, peut parallèlement :
<br><br>
+ envoyer un nombre fini de messages à d’autres acteurs.<br>
+ créer un nombre fini de nouveaux acteurs.<br>
+ spécifier le comportement à avoir lors de la prochaine réception de messages.<br>
<br/>
L’exécution des tâches ci-dessus n’est pas ordonnée, elles peuvent être parallélisées.
<br/><br/>
L’avancée fondamentale du modèle d’acteur est qu’il découple l’émetteur du message du message lui-même, permettant donc l’asynchronisme des communications et l’introduction de structures de contrôle dédiées à l’échange de messages.
<br/><br/>
Les destinataires des messages sont identifiés à l’aide d’adresses. Un acteur doit connaître l’adresse de l’acteur à qui il veut envoyer un message. Les adresses des acteurs créés sont connues de l’acteur parent. Les adresses peuvent être échangées par message.
<br/><br/>
Du fait de l’asynchronisme des communications, de la création dynamique d’acteurs et de l’échange des adresses des acteurs, le modèle est intrinsèquement asynchrone.
</i></p>
<p>
<a href="http://fr.wikipedia.org/wiki/Mod%C3%A8le_d%27acteur" target="_blank">http://fr.wikipedia.org/wiki/Mod%C3%A8le_d%27acteur</a>
</p>

<p>
Les actor peuvent réduire les problèmes de synchronisation que l'on peut avoir en utilisant des structures de données partagées. Pour ce faire, chaque actor utilise sa propre copie de donnée. Chaque actor peut envoyer un message à un autre actor ou travailler sur un message. Il n'est donc pas necessaire de se soucié des "lock" car chaque message possède ses propres données.
</p>
<p>
J'espère que vous avez compris ce que j'essais d'expliquer, peut être qu'un diagramme pourrait aider.
</p>


## Appplication MultiThread utilisant des données partagées

La plupart du temps, on crée plusieurs thread pour accélérer les traitements, mais ceci introduit le fait de "locker" les données entre thread. Un delai est donc creer pour attendre qu'une donnée soit "dé-locker" pour être utilisée. 


<br/>
<br/>
<img src="https://raw.githubusercontent.com/zeromq/netmq/master/docs/Images/ActorTrad.png"/>



Voici un exemple illustrant cette idée. Imaginez que nous ayons une structure représentant un compte bancaire.


    public class Account
    {
        public Account()
        {

        }

        public Account(int id, string name,
            string sortCode, decimal balance)
        {
            Id = id;
            Name = name;
            SortCode = sortCode;
            Balance = balance;
        }

        public int Id { get; set; }
        public string Name { get; set; }
        public string SortCode { get; set; }
        public decimal Balance { get; set; }

        public override string ToString()
        {
            return string.Format("Id: {0}, Name: {1}, SortCode: {2}, Balance: {3}", 
                Id, Name, SortCode, Balance);
        }
    }

Regardons le code Multithread utilisé ici...


    using System;
    using System.Collections.Generic;
    using System.Linq;
    using System.Text;
    using System.Threading;
    using System.Threading.Tasks;

    namespace SharedStateNeedForActors
    {
        class Program
        {
            private object syncLock = new object();
            private Account clientBankAccount;
            public Program()
            {
                clientBankAccount = new Account(1, "sacha barber", "112233", 0);
            }

            public async Task Run()
            {
                try
                {
                    await Task.Run(() =>
                    {
                        Console.WriteLine("Tread Id {0}, Account balance before: {1}",
                            Thread.CurrentThread.ManagedThreadId, clientBankAccount.Balance);

                        lock (syncLock)
                        {
                            Console.WriteLine("Tread Id {0}, Adding 10 to balance",
                               Thread.CurrentThread.ManagedThreadId);
                            clientBankAccount.Balance += 10;
                            Console.WriteLine("Tread Id {0}, Account balance before: {1}",
                                Thread.CurrentThread.ManagedThreadId, clientBankAccount.Balance);
                        }
                    });

                    await Task.Run(() =>
                    {
                        Console.WriteLine("Tread Id {0}, Account balance before: {1}",
                            Thread.CurrentThread.ManagedThreadId, clientBankAccount.Balance);
                        lock (syncLock)
                        {
                            Console.WriteLine("Tread Id {0}, Subtracting 4 to balance",
                               Thread.CurrentThread.ManagedThreadId);
                            clientBankAccount.Balance -= 4;
                            Console.WriteLine("Tread Id {0}, Account balance before: {1}",
                                Thread.CurrentThread.ManagedThreadId, clientBankAccount.Balance);
                        }
                    });
                }
                catch (Exception e)
                {
                    Console.WriteLine(e);
                }

            }

            static void Main(string[] args)
            {
                Program p = new Program();
                p.Run().Wait();
                Console.ReadLine();
            }
        }
    }

Bien que cet exemple ne représente pas une application réelle (qui écrirait une application qui crédite un compte dans un thread et le débite dans un autre ?), il marche quand même comme le montre le résultat :

<p><i>
Thread Id 6, Account balance before: 0<br/>
Thread Id 6, Adding 10 to balance<br/>
Thread Id 6, Account balance before: 10<br/>
Thread Id 10, Account balance before: 10<br/>
Thread Id 10, Subtracting 4 to balance<br/>
Thread Id 10, Account balance before: 6<br/>
</i></p>

Il doit quand même y avoir une meilleure manière de faire !


## Actor model

Le model actor a une approche différente. Le message est serializer entre les différents thread, il n y a donc aucune données partagées entre les thread.
L'idée est que chaque thread parle a un actor et reçois/envois un message avec un actor.

<br/>
<br/>
<img src="https://raw.githubusercontent.com/zeromq/netmq/master/docs/Images/ActorPass.png"/>


## Actor demo

Utilisons le même exemple que precedemment mais en version Actor.

Quelques classes helper...

**AccountActioner**

    public enum TransactionType { Debit = 1, Credit = 2 }

    public class AccountAction
    {
        public AccountAction()
        {

        }

        public AccountAction(TransactionType transactionType, decimal amount)
        {
            TransactionType = transactionType;
            Amount = amount;
        }

        public TransactionType TransactionType { get; set; }
        public decimal Amount { get; set; }
    }



**Account (same as before)**

    public class Account
    {
        public Account()
        {

        }

        public Account(int id, string name,
            string sortCode, decimal balance)
        {
            Id = id;
            Name = name;
            SortCode = sortCode;
            Balance = balance;
        }

        public int Id { get; set; }
        public string Name { get; set; }
        public string SortCode { get; set; }
        public decimal Balance { get; set; }


        public override string ToString()
        {
            return string.Format("Id: {0}, Name: {1}, SortCode: {2}, Balance: {3}", 
                Id, Name, SortCode, Balance);
        }
    }

Voici le code pour un <code>Actor</code> qui execute des action sur un <code>Account</code>. Cet exemple est volontairement simple, nous ne faisons que debiter et crediter un <code>Account</code> par un montant.


    using System;
    using System.Collections.Generic;
    using System.Linq;
    using System.Text;
    using System.Threading.Tasks;
    using NetMQ;
    using NetMQ.Actors;
    using NetMQ.InProcActors;
    using NetMQ.Sockets;
    using Newtonsoft.Json;

    namespace Actors
    {
        public class AccountActioner 
        {
            public class ShimHandler : IShimHandler<object>
            {
                private readonly NetMQContext context;
                private PairSocket shim;
                private Poller poller;
     
                public ShimHandler(NetMQContext context)
                {
                    this.context = context;
                }

                public void Initialise(object state)
                {

                }

                public void RunPipeline(PairSocket shim)
                {
                    this.shim = shim;
                    shim.ReceiveReady += OnShimReady;
                    shim.SignalOK();

                    poller = new Poller();
                    poller.AddSocket(shim);
                    poller.Start();

                }

           

                private void OnShimReady(object sender, NetMQSocketEventArgs e)
                {

                    string command = e.Socket.ReceiveString();

                    switch (command)
                    {
                        case ActorKnownMessages.END_PIPE:
                            Console.WriteLine("Actor received END_PIPE message");
                            poller.Stop(false);
                            break;
                        case "AmmendAccount":
                            Console.WriteLine("Actor received AmmendAccount message");
                            string accountJson = e.Socket.ReceiveString();
                            Account account = JsonConvert.DeserializeObject<Account>(accountJson);
                            string accountActionJson = e.Socket.ReceiveString();
                            AccountAction accountAction = JsonConvert.DeserializeObject<AccountAction>(accountActionJson);
                            Console.WriteLine("Incoming Account details are");
                            Console.WriteLine(account);
                            AmmendAccount(account, accountAction);
                            shim.Send(JsonConvert.SerializeObject(account));
                            break;
                    }

                }


                private void AmmendAccount(Account account, AccountAction accountAction)
                {
                    switch (accountAction.TransactionType)
                    {
                        case TransactionType.Credit:
                            account.Balance += accountAction.Amount;
                            break;
                        case TransactionType.Debit:
                            account.Balance -= accountAction.Amount;
                            break;
                    }
                }
            }

            private Actor<object> actor;
            private readonly NetMQContext context;

            public AccountActioner(NetMQContext context)
            {
                this.context = context;
            }

            public void Start()
            {
                if (actor != null)
                    return;

                actor = new Actor<object>(context, new ShimHandler(context), null);
            }

            public void Stop()
            {
                if (actor != null)
                {
                    actor.Dispose();
                    actor = null;
                }
            }

            public void SendPayload(Account account, AccountAction accountAction)
            {
                if (actor == null)
                    return;

                Console.WriteLine("About to send person to Actor");


                NetMQMessage message = new NetMQMessage();
                message.Append("AmmendAccount");
                message.Append(JsonConvert.SerializeObject(account));
                message.Append(JsonConvert.SerializeObject(accountAction));
                actor.SendMessage(message);
                
            }


            public Account GetPayLoad()
            {
                return JsonConvert.DeserializeObject<Account>(actor.ReceiveString());    
            }

        }
    }



    using System;
    using System.Collections.Generic;
    using System.Linq;
    using System.Text;
    using System.Threading.Tasks;
    using NetMQ;

    namespace Actors
    {
        class Program
        {
            static void Main(string[] args)
            {

                var context = NetMQContext.Create();

                //CommandActioner uses an NetMq.Actor internally
                AccountActioner accountActioner = new AccountActioner(context);


                Account clientBankAccount = new Account(1, "Doron Semech", "112233", 0);
                PrintAccount(clientBankAccount);

                accountActioner.Start();
                Console.WriteLine("Sending account to AccountActioner/Actor");
                accountActioner.SendPayload(clientBankAccount,
                    new AccountAction(TransactionType.Credit, 15));


                clientBankAccount = accountActioner.GetPayLoad();
                PrintAccount(clientBankAccount);


                accountActioner.Stop();
                Console.WriteLine();
                Console.WriteLine("Sending account to AccountActioner/Actor");
                accountActioner.SendPayload(clientBankAccount,
                    new AccountAction(TransactionType.Credit, 15));
                PrintAccount(clientBankAccount);

                Console.ReadLine();
            }

            static void PrintAccount(Account account)
            {
                Console.WriteLine("Account now");
                Console.WriteLine(account);
                Console.WriteLine();

            }
        }
    }


Lorsque vous lancez le programme vous obenez :


<br/>
<br/>
<img src="https://raw.githubusercontent.com/zeromq/netmq/master/docs/Images/ActorsOut.png"/>




