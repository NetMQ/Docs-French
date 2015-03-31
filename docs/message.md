Qu'est ce qu'un Message
=====

Si vous arrivez là après avoir lu l'introduction ou vu quelques exemples, ce genre de code ne vous est pas inconnu...

    using (NetMQContext ctx = NetMQContext.Create())
    {
        using (var server = ctx.CreateResponseSocket())
        {
            server.Bind("tcp://127.0.0.1:5556");
            using (var client = ctx.CreateRequestSocket())
            {
                client.Connect("tcp://127.0.0.1:5556");

                client.Send("Hello");

                string fromClientMessage = server.ReceiveString();

                Console.WriteLine("From Client: {0}", fromClientMessage);

                server.Send("Hi Back");

                string fromServerMessage = client.ReceiveString();

                Console.WriteLine("From Server: {0}", fromServerMessage);

                Console.ReadLine();
            }
        }
    }

Ici c'est la méthode <code>RecieveString()</code> des <code>NetMQSocket</code> qui est utilisée. Cette méthode est utile mais a ses limites car ZeroMQ (et donc NetMQ) est basé sur le concept de 'Frame', ce qui induit un protocole dans les échanges de données suivant les NetMqSocket utilisées.

Par exemple, la [RouterSocket](http://netmq.readthedocs.org/fr/latest/routerDealer/) utilise un certain protocole (ordre et contenu de frame) pour communiquer. Les différentes frame sont "empilés" et dépilés par les NetMQSocket. Une des frame de la NetMQSocket Router contiendra l'adress Ip du client afin que le router puisse envoyer une réponse à celui-ci

Vous pouvez construire votre propre protocole en utilisant les frames, par exemple :

+ Vous pouvez décider d'utiliser la frame[0] comme étant un message spécifique définissant le type de message qui va suivre,
évitant a une socket de lire la suite si elle n'est pas intéréssé par ce type (exemple Pub/Sub avec Topic)
+ Vous pouvez décider d'utiliser la frame[0] comme étant une commande, la Frame[1] comme paramètre et la frame[2] un conteneur de donnée (Objet serializé en json par exemple)

Ce sont juste quelques exemples et vous pouvez utiliser les frames comme bon vous sembles.

Quand vous travaillez avec des messages en plusieurs partie (frame) vous devez envoyez/recevoir toutes les parties du message avec lesquelles vous voulez travailler.

Il y a aussi le concept de 'more' sur les messages, nous en parlerons plus bas.


## Commen creer des Frames

Créer un message composé de plusieurs parties est très simple. Il suffit d'utiliser la classe <code>NetMQMessage</code>, et d'utiliser l'une des nombreuses surcharge de la méthode <code>Append()</code>. (des surcharges sont faites pour gérer les Blob/NetMQFrame/Byte[]/int/long/string)

Voici un exemple ou nous créons un <code>NetMQMessage</code> qui contient 2 <code>NetMQFrame</code>(s) et nous utilisons 
la méthode <code>NetMQMessage.Append()</code> pour mettre 2 string dans ces frames.

    var message = new NetMQMessage();
    message.Append("IAmFrame0");
    message.Append("IAmFrame1");
    server.SendMessage(message);

Il y a une autre manière de faire en utilisant la méthode <code>IOutgoingSocket.SendMore()</code>. Cette méthode n'a pas autant de surcharge que <code>NetMQMessage.Append()</code> mais permet d'envoyer des Byte[] et des String.

Voici un exemple d'utilisation de <code>IOutgoingSocket.SendMore()</code>


    var client = ctx.CreateRequestSocket()
    client.Connect("tcp://127.0.0.1:5556");

    //client send message
    client.SendMore("A");
    client.Send("Hello");

Attention, en utilisant <code>IOutgoingSocket.SendMore()</code> vous devez appeler <code>IOutgoingSocket.Send()</code> pour la dernière partie de votre message. Si vous utilisez la méthode <code>SendMessage()</code>, ceci est géré pour vous.



## Comment lire des Frames

Lire plusieurs Frames peut être fait de deux manières. Vous pouvez utiliser la méthode <code>RecieveString(out more)</code> plusieurs fois (vous devez gérer le fait de savoir combien il y a de frames dans le message)."out more" vous renverra true si d'autre frames sont disponibles.

Par exemple : 

    //client send message
    client.SendMore("A");
    client.Send("Hello");

    //server receive 1st part
    bool more;
    string messagePart1 = server.ReceiveString(out more);
    Console.WriteLine("string messagePart1 = server.ReceiveString(out more)");
    Console.WriteLine("messagePart1={0}", messagePart1);
    Console.WriteLine("HasMore={0}", more);


    //server receive 2nd part
    if (more)
    {
        string messagePart2 = server.ReceiveString(out more);
        Console.WriteLine("string messagePart2 = server.ReceiveString(out more)");
        Console.WriteLine("messagePart1={0}", messagePart2);
        Console.WriteLine("HasMore={0}", more);
        Console.WriteLine("================================");
    }
    
Une méthode plus simple est d'utiliser <code>RecieveMessage()</code>, et ensuite de lire les frames du message.
Voici le même exemple avec <code>RecieveMessage()</code>:


    var message4 = client.ReceiveMessage();
    Console.WriteLine("message4={0}", message4);
    Console.WriteLine("message4.FrameCount={0}", message4.FrameCount);

    Console.WriteLine("message4[0]={0}", message4[0].ConvertToString());
    Console.WriteLine("message4[1]={0}", message4[1].ConvertToString());



## Un exemple complet



    using System;
    using NetMQ;

    namespace ConsoleApplication2
    {
        internal class Program
        {
            private static void Main(string[] args)
            {
                using (NetMQContext ctx = NetMQContext.Create())
                {
                    using (var server = ctx.CreateResponseSocket())
                    {
                        server.Bind("tcp://127.0.0.1:5556");

                        using (var client = ctx.CreateRequestSocket())
                        {

                            client.Connect("tcp://127.0.0.1:5556");

                            //client send message
                            client.SendMore("A");
                            client.Send("Hello");

                            //server receive 1st part
                            bool more;
                            string messagePart1 = server.ReceiveString(out more);
                            Console.WriteLine("string messagePart1 = server.ReceiveString(out more)");
                            Console.WriteLine("messagePart1={0}", messagePart1);
                            Console.WriteLine("HasMore={0}", more);


                            //server receive 2nd part
                            if (more)
                            {
                                string messagePart2 = server.ReceiveString(out more);
                                Console.WriteLine("string messagePart2 = server.ReceiveString(out more)");
                                Console.WriteLine("messagePart1={0}", messagePart2);
                                Console.WriteLine("HasMore={0}", more);
                                Console.WriteLine("================================");
                            }


                            //server send message, this time use NetMqMessage
                            //which will be sent as frames if the client calls
                            //ReceieveMessage()
                            var m3 = new NetMQMessage();
                            m3.Append("From");
                            m3.Append("Server");
                            server.SendMessage(m3);
                            Console.WriteLine("Sending 2 frame message");




                            //client receive
                            var message4 = client.ReceiveMessage();
                            Console.WriteLine("message4={0}", message4);
                            Console.WriteLine("message4.FrameCount={0}", message4.FrameCount);

                            Console.WriteLine("message4[0]={0}", message4[0].ConvertToString());
                            Console.WriteLine("message4[1]={0}", message4[1].ConvertToString());


                            Console.ReadLine();
                        }
                    }
                }
            }
        }
    }


Résultat obtenu :

<p>
<i>
string messagePart1 = server.ReceiveString(out more)<br/>
messagePart1=A<br/>
HasMore=True<br/>
string messagePart2 = server.ReceiveString(out more)<br/>
messagePart1=Hello<br/>
HasMore=False<br/>
================================<br/>
Sending 2 frame message<br/>
message4=NetMQMessage[From,Server]<br/>
message4.FrameCount=2<br/>
message4[0]=From<br/>
message4[1]=Server<br/>


</i>
</p>

