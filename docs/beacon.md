Beacon
======

`NetMQBeacon` implemente un service de découverte de noeuds sur un réseau local.

Un "Beacon" peut broadcaster et/ou capturer des annonces de service en utilisant UDP sur le réseau local.
Vous pouvez définir le format de sortie de vos "Beacon" et définir un filtre qui valide les "Beacon" rentrant.
Les "Beacons" sont envoyés et reçus de manière asynchrone en arrière plan.

Nous pouvons utilisés le `NetMQBeacon` pour découvrir et se connecter au autres services NetMQ/CZMQ qui sont disponibles sur le réseau local, et cela de manière automatique, sans avoir besoins d'une configuration centrale. Il faut que votre infrastructure supporte le broadcast de packet UDP. La plupart des Fournisseurs Cloud ne les supportent pas.

Cette implémentation utilise IPV4, et est un portage du projet [zbeacon from czmq](https://github.com/zeromq/czmq#toc4-425).


## Example: Implémenter un Bus

Voici comment implémenter un simple bus qui utilise `NetMQBeacon`. Nous allons autoriser des noeuds a découvrir d'autre noeuds, simplement grâce à un partage de numéro de port UDP (chercher "broadcast UDP" sous google pour plus d'information)

* Chaque noeud du bus bind une socket SUB et se connecte aux autres noeuds avec une socket PUB.
* Chaque noeud va utiliser `NetMQBeacon` pour annoncer son existence et découvrir les autres noeuds. Nous allons utiliser `NetMQActor` pour implémenter nos noeuds.

### The Bus class

    :::csharp
    public class Bus
    {
        // Actor Protocol
        public const string PublishCommand = "P";

        // Dead nodes timeout
        private readonly TimeSpan DeadNodeTimeout = TimeSpan.FromSeconds(10);

        // we will use this to check if we already know about the node
        class NodeKey
        {
            public NodeKey(string name, int port)
            {
                Name = name;
                Port = port;
            }

            public string Name { get; private set; }
            public int Port { get; private set; }

            protected bool Equals(NodeKey other)
            {
                return string.Equals(Name, other.Name) && Port == other.Port;
            }

            public override bool Equals(object obj)
            {
                if (ReferenceEquals(null, obj)) return false;
                if (ReferenceEquals(this, obj)) return true;
                if (obj.GetType() != this.GetType()) return false;
                return Equals((NodeKey)obj);
            }

            public override int GetHashCode()
            {
                unchecked
                {
                    return ((Name != null ? Name.GetHashCode() : 0) * 397) ^ Port;
                }
            }
        }

        private readonly NetMQContext m_context;
        private readonly int m_broadcastPort;

        private NetMQActor m_actor;

        private PublisherSocket m_publisher;
        private SubscriberSocket m_subscriber;
        private NetMQBeacon m_beacon;
        private Poller m_poller;
        private PairSocket m_shim;
        private Dictionary<NodeKey, DateTime> m_nodes;

        private Bus(NetMQContext context, int broadcastPort)
        {
            m_nodes = new Dictionary<NodeKey, DateTime>();
            m_context = context;
            m_broadcastPort = broadcastPort;
            m_actor = NetMQActor.Create(context, RunActor);
        }

        /// <summary>
        /// Creates a new message bus actor. All communication with the bus is
        /// through the returned <see cref="NetMQActor"/>.
        /// </summary>
        public static NetMQActor Create(NetMQContext context, int broadcastPort)
        {
            Bus node = new Bus(context, broadcastPort);
            return node.m_actor;
        }

        private void RunActor(PairSocket shim)
        {
            // save the shim to the class to use later
            m_shim = shim;

            // create all subscriber, publisher and beacon
            using (m_subscriber = m_context.CreateSubscriberSocket())
            using (m_publisher = m_context.CreatePublisherSocket())
            using (m_beacon = new NetMQBeacon(m_context))
            {
                // listen to actor commands
                m_shim.ReceiveReady += OnShimReady;

                // subscribe to all messages
                m_subscriber.Subscribe("");

                // we bind to a random port, we will later publish this port
                // using the beacon
                int randomPort = m_subscriber.BindRandomPort("tcp://*");

                // listen to incoming messages from other publishers
                m_subscriber.ReceiveReady += OnSubscriberReady;

                // configure the beacon to listen on the broadcast port
                m_beacon.Configure(m_broadcastPort);

                // publishing the random port to all other nodes
                m_beacon.Publish(randomPort.ToString(), TimeSpan.FromSeconds(1));

                // Subscribe to all beacon on the port
                m_beacon.Subscribe("");

                // listen to incoming beacons
                m_beacon.ReceiveReady += OnBeaconReady;

                // Create and configure the poller with all sockets
                m_poller = new Poller(m_shim, m_subscriber, m_beacon);

                // Create a timer to clear dead nodes
                NetMQTimer timer = new NetMQTimer(TimeSpan.FromSeconds(1));
                timer.Elapsed += ClearDeadNodes;
                m_poller.AddTimer(timer);

                // signal the actor that we finished with configuration and
                // ready to work
                m_shim.SignalOK();

                // polling until cancelled
                m_poller.PollTillCancelled();
            }
        }

        private void OnShimReady(object sender, NetMQSocketEventArgs e)
        {
            // new actor command
            string command = m_shim.ReceiveString();

            // check if we received end shim command
            if (command == NetMQActor.EndShimMessage)
            {
                // we cancel the socket which dispose and exist the shim
                m_poller.Cancel();
            }
            else if (command == PublishCommand)
            {
                // it is a publish command
                // we just forward everything to the publisher until end of message
                NetMQMessage message = m_shim.ReceiveMessage();
                m_publisher.SendMessage(message);
            }
        }

        private void OnSubscriberReady(object sender, NetMQSocketEventArgs e)
        {
            // we got a new message from the bus
            // let's forward everything to the shim
            NetMQMessage message = m_subscriber.ReceiveMessage();
            m_shim.SendMessage(message);
        }

        private void OnBeaconReady(object sender, NetMQBeaconEventArgs e)
        {
            // we got another beacon
            // let's check if we already know about the beacon
            string nodeName;
            int port = Convert.ToInt32(m_beacon.ReceiveString(out nodeName));

            // remove the port from the peer name
            nodeName = nodeName.Replace(":" + m_broadcastPort, "");

            NodeKey node = new NodeKey(nodeName, port);

            // check if node already exist
            if (!m_nodes.ContainsKey(node))
            {
                // we have a new node, let's add it and connect to subscriber
                m_nodes.Add(node, DateTime.Now);
                m_publisher.Connect(string.Format("tcp://{0}:{1}", nodeName, port));
            }
            else
            {
                m_nodes[node] = DateTime.Now;
            }
        }

        private void ClearDeadNodes(object sender, NetMQTimerEventArgs e)
        {
            // create an array with the dead nodes
            var deadNodes = m_nodes.
                Where(n => DateTime.Now > n.Value + DeadNodeTimeout).
                Select(n => n.Key).ToArray();

            // remove all the dead nodes from the nodes list and disconnect
            // from the publisher
            foreach (var node in deadNodes)
            {
                m_nodes.Remove(node);
                m_publisher.Disconnect(
                    string.Format("tcp://{0}:{1}", node.Name, node.Port));
            }
        }
    }

### Node implementation

Un noeud sur le bus est définit:

    :::csharp
    using (NetMQContext context = NetMQContext.Create())
    {
        // create a bus using broadcast port 9999
        var actor = Bus.Create(context, 9999);

        // beacons publish every second, so wait a little longer than that to
        // let all the other nodes connect to our new node
        Thread.Sleep(1100);

        // publish a hello message
        // note we can use NetMQSocket send and receive extension methods
        actor.SendMore(Bus.PublishCommand).Send("Hello?");

        // receive messages from other nodes on the bus
        while (true)
        {
            string message = actor.ReceiveString();

            if (message == "Hello?")
            {
                // another node is saying hello
                Console.WriteLine(message);

                // send back a welcome message
                actor.SendMore(Bus.PublishCommand).Send("Welcome!");
            }
            else
            {
                // it's probably a welcome message
                Console.WriteLine(message);
            }
        }
    }

## En savoir plus

* [Resoudre le problème de découverte de noeuds](http://hintjens.com/blog:32)
