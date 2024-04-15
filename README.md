1 PARTE:
SOLICITAÇÃO.

package main

import (
	"fmt"
	"log"
	"net/http"
	"strings"
)

type Request struct {
	ID       int    `json:"id"`
	Message  string `json:"message"`
	Nature   string `json:"nature"`
	Gravity  string `json:"gravity"`
}

var requests []Request
var lastID int

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
	r.HandleFunc("/support/requests", func(w http.ResponseWriter, r *http.Request) {
		if r.Method != http.MethodGet {
			w.WriteHeader(http.StatusMethodNotAllowed)
			return
		}
		for _, req := range requests {
			fmt.Fprintf(w, "ID: %d, Message: %s, Nature: %s, Gravity: %s\n", req.ID, req.Message, req.Nature, req.Gravity)
		}
	})
	log.Fatal(http.ListenAndServe(":8080", r))
}

