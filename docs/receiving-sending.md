Recevoir / Envoyer
=====

Si vous avez lu la page d'[Introduction](http://netmq.readthedocs.org/en/latest/introduction/), vous avez déjà vu un exemple utilisant les méthodes <code>ReceiveString()</code> et <code>SendString()</code>, mais NetMQ vous permet d'envoyer autre chose que des String.

Regardons les options que nous propose NetMQ.



## Recevoir

Une <code>NetMQSocket</code> (qui est la classe dont tous les types de socket héritent) ne possède qu'une méthode <code>public virtual void Receive(ref Msg msg, SendReceiveOptions options)</code>. La plupart du temps, vous ne l'utiliserez pas, vous utiliserez plutôt l'une des extensions disponible pour <code>IReceivingSocket</code>.

Voici la liste de ces extensions de méthode. Si vous ne trouvez pas celle qu'il vous faut, vous pouvez écrire la vôtre.


    public static class ReceivingSocketExtensions
    {
        public static byte[] Receive(this IReceivingSocket socket);
        public static byte[] Receive(this IReceivingSocket socket, out bool hasMore);
        public static byte[] Receive(this IReceivingSocket socket, SendReceiveOptions options);
        public static byte[] Receive(this IReceivingSocket socket, bool dontWait, out bool hasMore);
        public static byte[] Receive(this IReceivingSocket socket, SendReceiveOptions options, out bool hasMore);
        public static NetMQMessage ReceiveMessage(this IReceivingSocket socket, bool dontWait = false);
        public static NetMQMessage ReceiveMessage(this NetMQSocket socket, TimeSpan timeout);
        public static void ReceiveMessage(this IReceivingSocket socket, NetMQMessage message, bool dontWait = false);
        public static IEnumerable<byte[]> ReceiveMessages(this IReceivingSocket socket);
        public static string ReceiveString(this IReceivingSocket socket);
        public static string ReceiveString(this IReceivingSocket socket, Encoding encoding);
        public static string ReceiveString(this IReceivingSocket socket, out bool hasMore);
        public static string ReceiveString(this IReceivingSocket socket, SendReceiveOptions options);
        public static string ReceiveString(this NetMQSocket socket, TimeSpan timeout);
        public static string ReceiveString(this IReceivingSocket socket, bool dontWait, out bool hasMore);
        public static string ReceiveString(this IReceivingSocket socket, Encoding encoding, out bool hasMore);
        public static string ReceiveString(this IReceivingSocket socket, Encoding encoding, SendReceiveOptions options);
        public static string ReceiveString(this IReceivingSocket socket, SendReceiveOptions options, out bool hasMore);
        public static string ReceiveString(this NetMQSocket socket, Encoding encoding, TimeSpan timeout);
        public static string ReceiveString(this IReceivingSocket socket, Encoding encoding, bool dontWait, out bool hasMore);
        public static string ReceiveString(this IReceivingSocket socket, Encoding encoding, SendReceiveOptions options, out bool hasMore);
        public static IEnumerable<string> ReceiveStringMessages(this IReceivingSocket socket);
        public static IEnumerable<string> ReceiveStringMessages(this IReceivingSocket socket, Encoding encoding);
        ....
        ....
    }


Voici un exemple d'implémentation d'une extension de méthode, ce qui devrait vous aider si vous décidez d'écrire la vôtre.


    public static string ReceiveString(this IReceivingSocket socket, Encoding encoding, SendReceiveOptions options, out bool hasMore)
    {
        Msg msg = new Msg();
        msg.InitEmpty();
        socket.Receive(ref msg, options);
        hasMore = msg.HasMore;
        string data = string.Empty;
        if (msg.Size > 0)
        {
            data = encoding.GetString(msg.Data, 0, msg.Size);
        }
        msg.Close();
        return data;
    }



## Envoyer

Une <code>NetMQSocket</code> (qui est la classe dont tous les types de socket héritent) ne possède qu'une méthode <code>public virtual void Send(ref Msg msg, SendReceiveOptions options)</code>. La plupart du temps, vous ne l'utiliserez pas, vous utiliserez plutôt l'une des extensions disponible pour <code>IOutgoingSocket</code>.

Voici la liste de ces extensions de méthode. Si vous ne trouvez pas celle qu'il vous faut, vous pouvez écrire la vôtre.



    public static class OutgoingSocketExtensions
    {
        public static void Send(this IOutgoingSocket socket, byte[] data);
        public static void Send(this IOutgoingSocket socket, byte[] data, int length, SendReceiveOptions options);
        public static void Send(this IOutgoingSocket socket, string message, bool dontWait = false, bool sendMore = false);
        public static void Send(this IOutgoingSocket socket, string message, Encoding encoding, SendReceiveOptions options);
        public static void Send(this IOutgoingSocket socket, byte[] data, int length, bool dontWait = false, bool sendMore = false);
        public static void Send(this IOutgoingSocket socket, string message, Encoding encoding, bool dontWait = false, bool sendMore = false);
        public static void SendMessage(this IOutgoingSocket socket, NetMQMessage message, bool dontWait = false);
        public static IOutgoingSocket SendMore(this IOutgoingSocket socket, byte[] data, bool dontWait = false);
        public static IOutgoingSocket SendMore(this IOutgoingSocket socket, string message, bool dontWait = false);
        public static IOutgoingSocket SendMore(this IOutgoingSocket socket, byte[] data, int length, bool dontWait = false);
        public static IOutgoingSocket SendMore(this IOutgoingSocket socket, string message, Encoding encoding, bool dontWait = false);
        ....
        ....
    }


Voici un exemple d'implémentation d'une extension de méthode, ce qui devrait vous aider si vous décidez d'écrire la vôtre.



    public static void Send(this IOutgoingSocket socket, string message, Encoding encoding, SendReceiveOptions options)
    {
        Msg msg = new Msg();
        msg.InitPool(encoding.GetByteCount(message));
        encoding.GetBytes(message, 0, message.Length, msg.Data, 0);
        socket.Send(ref msg, options);
        msg.Close();
    }


## Pour aller plus loin

Si vous regarder certaines signatures des méthodes et vous demandez pourquoi/comment les utiliser, vous devriez lire un peu plus sur la philosophie de Messaging que NetMQ utilise. La page [Message](http://netmq.readthedocs.org/en/latest/message/) vous donnera plus de détail.
