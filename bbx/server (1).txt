const WebSocket = require("ws");
const crypto = require("crypto");
const url = require("url"); // Dutangiye gukoresha 'url' kugirango dukureho parameters muri URL

const wss = new WebSocket.Server({ port: 8080 });
console.log("SERVER STARTED ON ws://localhost:8080");

// Stores mapping of active student details (ID -> {username, class})
// Igipimo kibika amakuru y'abanyeshuri bahuje screens (ID -> {username, class})
const studentUserInfo = new Map(); 

wss.on("connection", (ws, req) => {
    // Dukura username na class muri URL
    const params = url.parse(req.url, true).query;
    ws.role = params.role === "teacher" ? "teacher" : "student";
    
    if (ws.role === "student" && params.username && params.class) {
        // Generate a simple, unique ID for the session
        ws.studentId = crypto.randomBytes(4).toString('hex');
        
        // Bika amakuru y'umunyeshuri
        const userInfo = {
            id: ws.studentId,
            username: params.username,
            class: params.class
        };
        studentUserInfo.set(ws.studentId, userInfo);
        
        console.log(`New Student Connected: ${ws.studentId} (${userInfo.username} from ${userInfo.class})`);
        
        // 1. Oherereza amakuru yuzuye (INFO) ku munyeshuri.
        ws.send(`INFO:${JSON.stringify(userInfo)}`);

        // 2. Oherereza amakuru y'umunyeshuri mushya ku barimu bose.
        const teacherMessage = `NEW_STUDENT:${JSON.stringify(userInfo)}`;
        wss.clients.forEach(client => {
            if (client.readyState === WebSocket.OPEN && client.role === "teacher") {
                client.send(teacherMessage);
            }
        });
        
    } else if (ws.role === "teacher") {
        console.log("New Teacher Connected");
        // Iyo umwarimu ahanjye, twohereza urutonde rw'abanyeshuri bose bahuje screens
        const activeStudentsList = Array.from(studentUserInfo.values());
        ws.send(`ACTIVE_LIST:${JSON.stringify(activeStudentsList)}`);
    } else {
        console.log("Connection rejected: Missing username or class.");
        ws.close();
        return;
    }

    ws.on("message", (data) => {
        // Abanyeshuri gusa nibo bagomba kohereza data ya screen
        if (ws.role === "student" && ws.studentId) {
            
            // Dushinga buffer irimo ID y'umunyeshuri ikurikiwe na screen data.
            const idBuffer = Buffer.from(ws.studentId + "|", 'utf8');
            const messageBuffer = Buffer.concat([idBuffer, data]);

            // Twoherereza buffer ku barimu bose bahuje
            wss.clients.forEach(client => {
                if (client.readyState === WebSocket.OPEN && client.role === "teacher") {
                    client.send(messageBuffer);
                }
            });
        }
    });

    ws.on("close", () => {
        if (ws.role === "student" && ws.studentId) {
            studentUserInfo.delete(ws.studentId);
            console.log(`Student Disconnected: ${ws.studentId}`);
            
            // Twoherereza ubutumwa ku barimu ngo bakureho iyo screen
            const disconnectMessage = Buffer.from(`DISCONNECT:${ws.studentId}`, 'utf8');
             wss.clients.forEach(client => {
                if (client.readyState === WebSocket.OPEN && client.role === "teacher") {
                    client.send(disconnectMessage);
                }
            });
        }
    });
});