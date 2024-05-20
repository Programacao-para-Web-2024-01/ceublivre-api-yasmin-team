package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"os"
	"os/signal"
	"strconv"
	"strings"
	"sync"
	"syscall"
	"time"
)

type Request struct {
	ID      int    `json:"id"`
	Message string `json:"message"`
	Nature  string `json:"nature"`
	Gravity string `json:"gravity"`
}

type Nature struct {
	ID   int    `json:"id"`
	Name string `json:"name"`
}

var (
	requests    []Request
	requestsMux sync.RWMutex
	lastID      int
	natures     []Nature
	naturesMux  sync.RWMutex
	natureID    int
	shutdown    = make(chan os.Signal, 1)
)

func classifyRequest(message string) (string, string) {
	var nature, gravity string

	if strings.Contains(message, "erro") || strings.Contains(message, "falha") {
		nature = "Técnica"
	} else if strings.Contains(message, "reembolso") || strings.Contains(message, "pagamento") {
		nature = "Financeira"
	} else {
		nature = "Geral"
	}

	if strings.Contains(message, "urgente") || strings.Contains(message, "imediato") {
		gravity = "Alta"
	} else {
		gravity = "Normal"
	}

	return nature, gravity
}

func saveRequest(req Request) {
	requestsMux.Lock()
	defer requestsMux.Unlock()

	lastID++
	req.ID = lastID
	requests = append(requests, req)
}

func getRequests() []Request {
	requestsMux.RLock()
	defer requestsMux.RUnlock()

	return requests
}

func createNature(name string) {
	naturesMux.Lock()
	defer naturesMux.Unlock()

	natureID++
	n := Nature{
		ID:   natureID,
		Name: name,
	}
	natures = append(natures, n)
}

func listNatures() []Nature {
	naturesMux.RLock()
	defer naturesMux.RUnlock()

	return natures
}

func deleteNature(id int) {
	naturesMux.Lock()
	defer naturesMux.Unlock()

	for i, n := range natures {
		if n.ID == id {
			natures = append(natures[:i], natures[i+1:]...)
			return
		}
	}
}

func main() {
	r := http.NewServeMux()

	r.HandleFunc("/support/request", func(w http.ResponseWriter, r *http.Request) {
		if r.Method != http.MethodPost {
			http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
			return
		}

		var req Request
		if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
			http.Error(w, "Bad request", http.StatusBadRequest)
			return
		}

		// Classificar a solicitação com base na mensagem
		req.Nature, req.Gravity = classifyRequest(req.Message)
		saveRequest(req)

		fmt.Fprintf(w, "Request ID %d registered\n", req.ID)
	})


	r.HandleFunc("/support/requests", func(w http.ResponseWriter, r *http.Request) {
		if r.Method != http.MethodGet {
			http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
			return
		}

		requestList := getRequests()

		w.Header().Set("Content-Type", "application/json")
		if err := json.NewEncoder(w).Encode(requestList); err != nil {
			http.Error(w, "Internal server error", http.StatusInternalServerError)
			log.Printf("Error encoding response: %v\n", err)
			return
		}
	})

	r.HandleFunc("/support/natures", func(w http.ResponseWriter, r *http.Request) {
		switch r.Method {
		case http.MethodPost:
			var n Nature
			if err := json.NewDecoder(r.Body).Decode(&n); err != nil {
				http.Error(w, "Bad request", http.StatusBadRequest)
				return
			}
			createNature(n.Name)
			fmt.Fprintf(w, "Nature %s created\n", n.Name)
		case http.MethodGet:
			natureList := listNatures()
			w.Header().Set("Content-Type", "application/json")
			if err := json.NewEncoder(w).Encode(natureList); err != nil {
				http.Error(w, "Internal server error", http.StatusInternalServerError)
				log.Printf("Error encoding response: %v\n", err)
				return
			}
		default:
			http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
		}
	})

	r.HandleFunc("/support/natures/", func(w http.ResponseWriter, r *http.Request) {
		if r.Method != http.MethodDelete {
			http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
			return
		}

		parts := strings.Split(r.URL.Path, "/")
		if len(parts) != 4 {
			http.Error(w, "Bad request", http.StatusBadRequest)
			return
		}

		natureID, err := strconv.Atoi(parts[3])
		if err != nil {
			http.Error(w, "Bad request", http.StatusBadRequest)
			return
		}

		deleteNature(natureID)
		fmt.Fprintf(w, "Nature with ID %d deleted\n", natureID)
	})

	signal.Notify(shutdown, syscall.SIGINT, syscall.SIGTERM)

	server := &http.Server{
		Addr:    ":8081",
		Handler: r,
	}

	go func() {
		<-shutdown
		log.Println("Shutting down the server...")
		ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
		defer cancel()
		if err := server.Shutdown(ctx); err != nil {
			log.Fatalf("Error shutting down server: %v", err)
		}
	}()

	log.Println("Server started, press Ctrl+C to shutdown")
	if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
		log.Fatalf("Error starting server: %v", err)
	}
}





