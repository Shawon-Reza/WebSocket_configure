# WebSocket_configure


# Service 

```jsx 
import { getAuthData } from "../../../config/Config";

// ........................Get Auth Token.......................... //



// ------------------ **WebSocket Connection for Chat List** ------------------ //
export const connectWebSocketForChatList = ({ onMessage, onSeen }) => {

    let token = null;
    try {
        const { accessToken } = getAuthData();
        token = accessToken;
    } catch (e) {
        console.error("Failed to get auth data:", e);
        return null; // stop if token not found
    }
    if (!token) {
        console.error("No access token found, cannot connect WebSocket");
        return null;
    }

    const wsProtocol = window.location.protocol === "https:" ? "wss" : "ws";
    const wsUrl = `${wsProtocol}://10.10.13.2:8000/ws/rooms/?token=${token}`;
    const socket = new WebSocket(wsUrl);
    socket.onopen = () => console.log("✅ WebSocket connected for Chat List");


    socket.onmessage = (event) => {
        try {
            const data = JSON.parse(event.data);
            console.log("📩 WS message:", data);

            switch (data.type) {
                case "chat_message":
                    onMessage?.(data.message ?? data)
                    break

                case "messages_seen_update":
                    onSeen?.(data.message_ids, data.seen_by)
                    break

                default:
                    console.warn("Unknown WS event type:", data.type)
                    onMessage?.(data)
            }
        } catch (err) {
            console.error("Failed to parse WebSocket message:", err)
        }
    };


    socket.onclose = () =>
        console.log("❌ WebSocket disconnected for Chat List");
    socket.onerror = (e) => console.error("⚠️ WebSocket error", e);

    return socket;
};


```

# Connect the socket on specific component
```jsx

  const [chatList, setChatList] = useState([])
    const socketRef = useRef(null);

    useEffect(() => {
         socketRef.current = connectWebSocketForChatList({
            onMessage: (message) => {
                console.log("New chat message received:", message)
                setChatList((prevList) => [message, ...prevList])
            },
            onSeen: (messageIds, seenBy) => {
                console.log("Messages seen update:", messageIds, seenBy)
                setChatList((prevList) =>
                    prevList.map((chat) =>
                        messageIds.includes(chat.id)
                            ? { ...chat, seen_by: [...(chat.seen_by || []), ...seenBy] }
                            : chat
                    )
                )
            },
        })

        return () => {
            if (socketRef.current) {
                socketRef.current.close()
            }
        }
    }, [])
```

# Update new mast on cach on tanstack query
```jsx
 // ..............**Connecting to WebSocket**..................\\
  useEffect(() => {
    if (!chatRoom) return;

    socketRef.current = connectWebSocketForChat({
      roomId: chatRoom,

      onMessage: (message) => {
        console.log("New msg arives Chatpanel:", message)
        if (message.type !== 'message') return;

        const newMessage = message.data;
        // Updated messages and cache with new message
        queryClient.setQueryData(
          ['messages', chatRoom],
          (oldMessages = []) => {
            // Safety: ensure array
            if (!Array.isArray(oldMessages)) return oldMessages;

            // Prevent duplicate messages
            const alreadyExists = oldMessages.some(
              msg => msg.id === newMessage.id
            );

            if (alreadyExists) return oldMessages;

            // Add new message to TOP (newest first)
            return [newMessage, ...oldMessages];
          }
        );
      },
    });

    return () => {
      socketRef.current?.close();
    };
  }, [chatRoom, queryClient]);
```
