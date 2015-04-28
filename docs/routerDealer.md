Router Dealer
=====


## RouterSocket
<i>
La socket ROUTER, contrairement aux autres socket, traque chaques connections qu'elle a, et insert une frame d'identité au début de chaque message qu'elle recoit avant de passer le message. Une identité, parfois appellée adresse, est un "handle" unique pour la connection. Quand vous envoyez un message par une socket ROUTER, vous envoyez d'abord une frame d'identité.

Quand une socket ROUTER reçoit un message, elle ajoute l'identité du client dans le message et fait passer le message à l'application.Les messages reçus sont mis en file d'attente parmis tous les messages des clients connectés. Quand une socket ROUTER recçoit une réponse et renvoie cette réponse à un client, elle lit la première frame du message qui contient l'identité du client, retire cette frame du message et fait passer le message au client (elle connait l'identité grâce à cette première frame)

L'identité est un concept difficile à comprendre, mais c'est essentiel si vous voulez devenir un exprt ZeroMQ. La socket ROUTER invente une identité aléatoire pour chaque client avec lequel elle va travaillé. Si trois requètes REQ se connectent à une socket ROUTER, elle va creer trois identité aléatoire, une pour chaque REQ.
</i>
<br/>
<br/>
Ce texte est pris du <a href="http://zguide.zeromq.org/page:all" target="_blank">guide ZeroMQ</a>.

Disons qu'une socket <code>DealerSocket</code> a trois bytes d'identité ABC. Cela veut dire que la socket ROUTER contient une hashtable avec en clé ABC et en valeur l'adresse de la <code>DealerSocket</code>.

Quand nous recevons un message de la <code>DealerSocket</code>, nous avons trois frames.
<br/>
<br/>
<img src="https://github.com/imatix/zguide/raw/master/images/fig28.png"/>
</i>


### Identités et Adresses.
<i>
Le concept d'identité dans ZeroMQ fait référence aux socket ROUTER et comment elles identifient les connections qu'elles ont avec les autres sockets. Plus simplement, les identités sont utilisées comme adresses de retour des messages. La plupart du temps, l'identité est arbitraire et local à une socket ROUTER : C'est une clé dans une hashtable. Indépendemment, un noeud peut avoir une adresse physique comme "tcp://192.168.55.117:5670" ou logique (un Guid ou une adresse mail ou n'importe quelle clé unique)

Une application qui utilise une socket ROUTER pour parler à un client spécifique peut convertir une adresse logique en clé d'identité si elle a construite la hashtable nécéssaire. Comme une socket ROUTER annonce seulement l'identité de la connection spécifique d'un client quand ce client envoie un message, vous ne pouvez répondre qu'a un message et non directement vous connecter au client.

C'est aussi valable si vous inversez les rêgles et que vous forcez le ROUTER à se connecter à un client à la place d'attendre que le client se connecte. Cependant vous pouvez forcer la socket ROUTER a utiliser une adresse logique à la place d'une identité. La page de référence sur l'option "zmq_setsockopt" appelle cette option la socket identité. 
Cela marche comme cela : 

+ L'application client met l'option ZMQ_IDENTITY sur la socket cliente (DEALER or REQ) avant de binder ou de se connecter.
+ Habituellement le client se connecte à une socket ROUTER déjà attachéé. Mais la socket ROUTER peut aussi se connecter au client.
+ Lors de la connection,la socket cliente dit à la sockjet ROUTER "utilise cette identité pour cette connection".
+ Si la socket cliente ne le spécifie pas, alors la socket ROUTER utilise le mécanisme habituel de génération aléatoire d'identité.
+ La socket ROUTER utilise maintenant cette adresse logique comme identité du client pour chaque message venant de ce client.
+ La socket ROUTER s'attend aussi à recevoir cette première frame d'identité pour chaque message retour à retransmettre au client.
</i>
<br/>
<br/>
Le texte ci dessus est tiré du <a href="http://zguide.zeromq.org/page:all#Identities-and-Addresses" target="_blank">guide ZeroMQ, Identités et Adresses</a>

La socket ROUTER est asynchrone (non bloquante).




## DealerSocket

La socket <code>DealerSocket</code> ne fait rien de plus que la socket REQ, mise à part qu'elle est asynchrone (non bloquante).
Pour les autres socket, les méthodes <code>ReceieveXXX</code> / <code>SendXXX</code> sont  bloquantes et renverront des exceptions si vous n'appellez pas les choses dans le bon ordres ou plusieurs fois d'affillé.

Typiquement, une <code>DealerSocket</code> sera utilisée en conjonction d'une socket <code>RouterSocket</code>, c'est pourquoi nous avons décrite ces deux socket dans la même page de documentation.

Si vous voulez en savoir plus sur les combinaisons possibles avec la <code>DealerSocket</code>, regardez dans le guide ZeroMQ à la page <a href="http://zguide.zeromq.org/page:all#toc58" target="_blank">Request-Reply Combinations</a>.




## An example

Time for an example. The best way to think of this example is summarized in the bullet points below

+ There is one server. Which is a <code>RouterSocket</code>. Where the <code>RouterSocket</code>, will use the incoming clients socket (<code>DealerSocket</code>) identity, to work out how to route back the response message to the correct client socket
+ There are multiple clients created, each in its own thread. These clients are <code>DealerSocket</code>(s). The client socket will provide a fixed identity, such that the server (<code>RouterSocket</code>) wille be able to use the identity supplied to correctly route back messages for this client

Ok so that is the overview, what does the code look like, lets see:


    using System;
    using System.Text;
    using System.Threading;
    using System.Threading.Tasks;
    using NetMQ;
    using NetMQ.Sockets;

    namespace RouterDealer
    {
        public class Program
        {
            public void Run()
            {

                //NOTES
                //1. Use ThreadLocal<DealerSocket> where each thread has
                //  its own client DealerSocket to talk to server
                //2. Each thread can send using it own socket
                //3. Each thread socket is added to poller

                ThreadLocal<DealerSocket> clientSocketPerThread =
                    new ThreadLocal<DealerSocket>();
                int delay = 3000;
                Poller poller = new Poller();

                using (NetMQContext ctx = NetMQContext.Create())
                {
                    using (var server = ctx.CreateRouterSocket())
                    {
                        server.Bind("tcp://127.0.0.1:5556");

                        //start some threads, each with its own DealerSocket
                        //to talk to the server socket. Creates lots of sockets, 
                        //but no nasty race conditions no shared state, each 
                        //thread has its own socket, happy days
                        for (int i = 0; i < 3; i++)
                        {
                            Task.Factory.StartNew((state) =>
                            {
                                DealerSocket client = null;

                                if (!clientSocketPerThread.IsValueCreated)
                                {
                                    client = ctx.CreateDealerSocket();
                                    client.Options.Identity = Encoding.Unicode.GetBytes(state.ToString());
                                    client.Connect("tcp://127.0.0.1:5556");
                                    client.ReceiveReady += Client_ReceiveReady;
                                    clientSocketPerThread.Value = client;
                                    poller.AddSocket(client);
                                }
                                else
                                {
                                    client = clientSocketPerThread.Value;
                                }

                                while (true)
                                {
                                    var messageToServer = new NetMQMessage();
                                    messageToServer.AppendEmptyFrame();
                                    messageToServer.Append(state.ToString());
                                    Console.WriteLine("======================================");
                                    Console.WriteLine(" OUTGOING MESSAGE TO SERVER ");
                                    Console.WriteLine("======================================");
                                    PrintFrames("Client Sending", messageToServer);
                                    client.SendMessage(messageToServer);
                                    Thread.Sleep(delay);
                                }

                            }, string.Format("client {0}", i), TaskCreationOptions.LongRunning);
                        }

                        //start the poller
                        Task task = Task.Factory.StartNew(poller.Start);

                        //server loop
                        while (true)
                        {
                            var clientMessage = server.ReceiveMessage();
                            Console.WriteLine("======================================");
                            Console.WriteLine(" INCOMING CLIENT MESSAGE FROM CLIENT ");
                            Console.WriteLine("======================================");
                            PrintFrames("Server receiving", clientMessage);
                            if (clientMessage.FrameCount == 3)
                            {
                                var clientAddress = clientMessage[0];
                                var clientOriginalMessage = clientMessage[2].ConvertToString();
                                string response = string.Format("{0} back from server {1}",
                                    clientOriginalMessage, DateTime.Now.ToLongTimeString());
                                var messageToClient = new NetMQMessage();
                                messageToClient.Append(clientAddress);
                                messageToClient.AppendEmptyFrame();
                                messageToClient.Append(response);
                                server.SendMessage(messageToClient);
                            }
                        }
                    }
                }
            }

            void PrintFrames(string operationType, NetMQMessage message)
            {
                for (int i = 0; i < message.FrameCount; i++)
                {
                    Console.WriteLine("{0} Socket : Frame[{1}] = {2}", operationType, i,
                        message[i].ConvertToString());
                }
            }

            void Client_ReceiveReady(object sender, NetMQSocketEventArgs e)
            {
                bool hasmore = false;
                e.Socket.Receive(out hasmore);
                if (hasmore)
                {
                    string result = e.Socket.ReceiveString(out hasmore);
                    Console.WriteLine("REPLY {0}", result);
                }
            }

            [STAThread]
            public static void Main(string[] args)
            {
                Program p = new Program();
                p.Run();
            }
        }
    }


When you run this you should see some output something like this (remember this is asynchronous code here, so things may not come in the order you logically expect)


<i>
======================================<br/>
 OUTGOING MESSAGE TO SERVER<br/>
======================================<br/>
======================================<br/>
 OUTGOING MESSAGE TO SERVER<br/>
======================================<br/>
Client Sending Socket : Frame[0] =<br/>
Client Sending Socket : Frame[1] = client 1<br/>
Client Sending Socket : Frame[0] =<br/>
Client Sending Socket : Frame[1] = client 0<br/>
======================================<br/>
 INCOMING CLIENT MESSAGE FROM CLIENT<br/>
======================================<br/>
Server receiving Socket : Frame[0] = c l i e n t   1<br/>
Server receiving Socket : Frame[1] =<br/>
Server receiving Socket : Frame[2] = client 1<br/>
======================================<br/>
 INCOMING CLIENT MESSAGE FROM CLIENT<br/>
======================================<br/>
Server receiving Socket : Frame[0] = c l i e n t   0<br/>
Server receiving Socket : Frame[1] =<br/>
Server receiving Socket : Frame[2] = client 0<br/>
REPLY client 1 back from server 08:05:56<br/>
REPLY client 0 back from server 08:05:56<br/>
</i>
