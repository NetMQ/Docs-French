Request / Response
=====

Request / Response est la plus simple des combinaisons pour les socket NetMQ. Cela ne veut pas dire que <code>RequestSocket/ResponseSocket</code> doivent toujours être utilisées ensembles, il y a d'autres combinaisons possibles. Mais c'est une pattern qui est souvent utilisée.
<br/>
<br/>
Vous pouvez trouvez les différentes combinaisons de socket dans le <a href="http://zguide.zeromq.org/page:all" target="_blank">guide ZeroMQ</a>. Bien qu'il soit très simple de vous dire d'aller lire de la documentation ailleurs, la raison est qu'il n y a pas meilleur documentation que le guide. 

Mais continuons sur les Request/Response...




## Qu'est ce que Request/Response

La pattern Request / Response est la plus simple. Elle s'apparente à une requete web : vous faite une requete, vous recevez une réponse.

<code>RequestSocket / ResponseSocket</code> sont **synchrone, et donc bloquante**, vous génèrerez une exception si vous essayer de lire plus de message qu'il y en a.

La manière de travailler avec <code>RequestSocket/ResponseSocket</code> est la suivante :

1. Envoyez un message avec <code>RequestSocket</code>
2. Une <code>ResponseSocket</code>lit le message.
3. Cette même <code>ResponseSocket</code> envoi le message de réponse
4. La <code>RequestSocket</code> de l'étape 1 reçoie la réponse.

Voici un exemple ou les <code>RequestSocket/ResponseSocket</code> sont dans un même processus, mais cela est facilement découpable en module client/serveur.

Exemple:

    using System;
    using System.Collections.Generic;
    using System.Linq;
    using System.Text;
    using System.Threading;
    using System.Threading.Tasks;
    using NetMQ;

    namespace RequestResponse
    {
        class Program
        {
            static void Main(string[] args)
            {
                using (var context = NetMQContext.Create())
                {
                    using (var responseSocket = context.CreateResponseSocket())
                    {
                        responseSocket.Bind("tcp://*:5555");

                        using (var requestSocket = context.CreateRequestSocket())
                        {
                            requestSocket.Connect("tcp://localhost:5555");
                            Console.WriteLine("requestSocket : Sending 'Hello'");
                            requestSocket.Send("Hello");

                            var message = responseSocket.ReceiveString();

                            Console.WriteLine("responseSocket : Server Received '{0}'", message);

                            Console.WriteLine("responseSocket Sending 'World'");
                            responseSocket.Send("World");

                            message = requestSocket.ReceiveString();
                            Console.WriteLine("requestSocket : Received '{0}'", message);

                            Console.ReadLine();
                        }

                    }
                }
            }
        }
    }


Si vous lancez le code vous devriez obtenir le résultat suivant :
<br/>
<br/>
<br/>
<img src="https://raw.githubusercontent.com/zeromq/netmq/master/docs/Images/RequestResponse.png"/>





## Request/Response sont bloquante

Comme vu précedemment, <code>RequestSocket/ResponseSocket</code> sont bloquantes, ce qui veut dire que des appels inattendus <code>SendXXXX()</code> / <code>ReceiveXXXX()</code> vons généré des exceptions.

Dans cet exemple , nous appelons deux fois <code>Send()</code> dans la <code>RequestSocket</code>

<br/>
<br/>
<img src="https://raw.githubusercontent.com/zeromq/netmq/master/docs/Images/RequestResponse2Sends.png"/>




Dans cet exemple, nous appelons deux fois <code>RecieveString()</code> mais il n y a eu qu'un message envoyé par <code>RequestSocket</code>


<br/>
<br/>
<img src="https://raw.githubusercontent.com/zeromq/netmq/master/docs/Images/RequestResponse2Receives.png"/>


Il faut donc faire attention à la manière d'utiliser la pattern Request/Response.

