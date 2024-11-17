```
package main

import (
	"bufio"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"os"
	"os/exec"
	"path/filepath"
	"strings"
	"sync"

	"github.com/gorilla/websocket"
)

var upgrader = websocket.Upgrader{
	CheckOrigin: func(r *http.Request) bool {
		return true
	},
}

type CommandOutput struct {
	mu      sync.Mutex
	clients map[*websocket.Conn]bool
}

func newCommandOutput() *CommandOutput {
	return &CommandOutput{
		clients: make(map[*websocket.Conn]bool),
	}
}

func (co *CommandOutput) broadcast(msg string) {
	co.mu.Lock()
	defer co.mu.Unlock()
	for client := range co.clients {
		if err := client.WriteMessage(websocket.TextMessage, []byte(msg)); err != nil {
			client.Close()
			delete(co.clients, client)
		}
	}
}

var commandOutput = newCommandOutput()

// Handle WebSocket connections
func handleWebSocket(w http.ResponseWriter, r *http.Request) {
	conn, err := upgrader.Upgrade(w, r, nil)
	if err != nil {
		log.Println("WebSocket upgrade error:", err)
		return
	}
	commandOutput.mu.Lock()
	commandOutput.clients[conn] = true
	commandOutput.mu.Unlock()
}

// Handle the file list API
func handleFileList(w http.ResponseWriter, r *http.Request) {
	dir := "/root/"
	type FileInfo struct {
		Name string `json:"name"`
		URL  string `json:"url"`
	}
	files := []FileInfo{}

	entries, err := os.ReadDir(dir)
	if err != nil {
		log.Println("Error reading directory:", err)
		w.WriteHeader(http.StatusInternalServerError)
		return
	}

	for _, entry := range entries {
		if !entry.IsDir() {
			filePath := fmt.Sprintf("/download?file=%s", entry.Name())
			files = append(files, FileInfo{Name: entry.Name(), URL: filePath})
		}
	}

	w.Header().Set("Content-Type", "application/json")
	jsonData, err := json.Marshal(files)
	if err != nil {
		log.Println("Error marshaling JSON:", err)
		w.WriteHeader(http.StatusInternalServerError)
		return
	}
	w.Write(jsonData)
}

// Handle file download
func handleDownload(w http.ResponseWriter, r *http.Request) {
	dir := "/root/"
	fileName := r.URL.Query().Get("file")

	if fileName == "" {
		http.Error(w, "File name is required", http.StatusBadRequest)
		return
	}

	filePath := filepath.Join(dir, fileName)
	if _, err := os.Stat(filePath); os.IsNotExist(err) {
		http.Error(w, "File not found", http.StatusNotFound)
		return
	}

	w.Header().Set("Content-Disposition", fmt.Sprintf("attachment; filename=\"%s\"", fileName))
	w.Header().Set("Content-Type", "application/octet-stream")
	http.ServeFile(w, r, filePath)
}

// Handle the scan command
func handleScan(w http.ResponseWriter, r *http.Request) {
	cmd := exec.Command("hp-scan")
	stdout, err := cmd.StdoutPipe()
	if err != nil {
		log.Println("Error creating stdout pipe:", err)
		return
	}

	err = cmd.Start()
	if err != nil {
		log.Println("Error starting command:", err)
		return
	}

	reader := bufio.NewReader(stdout)
	go func() {
		for {
			line, err := reader.ReadString('\n')
			if err != nil {
				if err.Error() != "EOF" {
					log.Println("Error reading stdout:", err)
				}
				break
			}
			commandOutput.broadcast(strings.TrimSpace(line))
		}
		commandOutput.broadcast("Scan complete.")
	}()

	cmd.Wait()
}

func main() {
	http.Handle("/", http.FileServer(http.Dir("./static")))
	http.HandleFunc("/files", handleFileList)
	http.HandleFunc("/download", handleDownload)
	http.HandleFunc("/scan", handleScan)
	http.HandleFunc("/ws", handleWebSocket)

	log.Println("Starting server at :8080...")
	log.Fatal(http.ListenAndServe(":8080", nil))
}

```
static/index.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Scan and File List</title>
    <style>
        body { font-family: Arial, sans-serif; }
        #fileList a { display: block; margin: 5px 0; }
    </style>
</head>
<body>
    <h1>Scanner and File List</h1>

    <button id="scanBtn">Start Scan</button>
    <div id="output" style="margin-top: 20px; border: 1px solid #ddd; padding: 10px; height: 150px; overflow-y: auto;"></div>

    <h2>Files in /root/</h2>
    <div id="fileList"></div>

    <script>
        const outputDiv = document.getElementById("output");
        const fileList = document.getElementById("fileList");

        // WebSocket connection
        const ws = new WebSocket("ws://" + location.host + "/ws");
        ws.onmessage = (event) => {
            const message = document.createElement("div");
            message.textContent = event.data;
            outputDiv.appendChild(message);
            outputDiv.scrollTop = outputDiv.scrollHeight;
        };

        // Start scan
        document.getElementById("scanBtn").addEventListener("click", () => {
            fetch("/scan");
        });

        // Refresh file list
        async function refreshFileList() {
            const response = await fetch("/files");
            const files = await response.json();
            fileList.innerHTML = ""; // Clear existing list
            files.forEach(file => {
                const link = document.createElement("a");
                link.href = file.url;
                link.textContent = file.name;
                link.download = file.name;
                fileList.appendChild(link);
            });
        }

        // Refresh file list every 5 seconds
        setInterval(refreshFileList, 5000);
        refreshFileList();
    </script>
</body>
</html>

```

```