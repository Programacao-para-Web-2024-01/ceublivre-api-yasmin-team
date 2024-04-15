1 PARTE:
SOLICITAÇÃO.



  package main

import (
	"context"
	"fmt"
	"io"
	"log"
	"net/http"
	"os"
	"os/signal"
	"strings"
	"syscall"
)

type Request struct {
	ID      int    `json:"id"`
	Message string `json:"message"`
	Nature  string `json:"nature"`
	Gravity string `json:"gravity"`
}

var (
	requests []Request
	lastID   int
	shutdown = make(chan os.Signal, 1)
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

func main() {
	r := http.NewServeMux()

	r.HandleFunc("/support/request", func(w http.ResponseWriter, r *http.Request) {
		if r.Method != http.MethodPost {
			w.WriteHeader(http.StatusMethodNotAllowed)
			return
		}
		var req Request
		if err := r.ParseForm(); err != nil {
			w.WriteHeader(http.StatusBadRequest)
			return
		}
		req.Message = r.FormValue("message")
		req.Nature, req.Gravity = classifyRequest(req.Message)
		lastID++
		req.ID = lastID
		requests = append(requests, req)
		fmt.Fprintf(w, "Request ID %d registered\n", req.ID)
		w.WriteHeader(http.StatusOK)
	})

	r.HandleFunc("/support/request/photo", func(w http.ResponseWriter, r *http.Request) {
		if r.Method != http.MethodPost {
			w.WriteHeader(http.StatusMethodNotAllowed)
			return
		}
		r.ParseMultipartForm(10 << 20) // Limit to 10MB files
		file, handler, err := r.FormFile("photo")
		if err != nil {
			fmt.Println("Error Retrieving the File")
			fmt.Println(err)
			return
		}
		defer file.Close()
		fmt.Printf("Uploaded File: %+v\n", handler.Filename)
		fmt.Printf("File Size: %+v\n", handler.Size)
		fmt.Printf("MIME Header: %+v\n", handler.Header)

		f, err := os.OpenFile("./uploads/"+handler.Filename, os.O_WRONLY|os.O_CREATE, 0666)
		if err != nil {
			fmt.Println(err)
			return
		}
		defer f.Close()
		io.Copy(f, file)
		fmt.Fprintf(w, "File uploaded successfully")
		w.WriteHeader(http.StatusOK)
	})

	r.HandleFunc("/support/requests", func(w http.ResponseWriter, r *http.Request) {
		if r.Method != http.MethodGet {
			w.WriteHeader(http.StatusMethodNotAllowed)
			return
		}
		for _, req := range requests {
			fmt.Fprintf(w, "ID: %d, Message: %s, Nature: %s, Gravity: %s\n", req.ID, req.Message, req.Nature, req.Gravity)
		}
	})

	signal.Notify(shutdown, syscall.SIGINT, syscall.SIGTERM)

	server := &http.Server{
		Addr:    ":8080",
		Handler: r,
	}

	go func() {
		<-shutdown
		log.Println("Shutting down the server...")
		if err := server.Shutdown(context.Background()); err != nil {
			log.Fatalf("Error shutting down server: %v", err)
		}
	}()

	log.Println("Server started, press Ctrl+C to shutdown")
	if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
		log.Fatalf("Error starting server: %v", err)
	}
}



