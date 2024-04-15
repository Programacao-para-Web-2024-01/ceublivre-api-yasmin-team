package main

import (
	"fmt"
	"log"
	"net/http"
)

type Request struct {
	ID      int    `json:"id"`
	Message string `json:"message"`
}

var requests []Request
var lastID int

func main() {
	r := http.NewServeMux()

	r.HandleFunc("/support/request", func(w http.ResponseWriter, r *http.Request) {
		if r.Method != http.MethodPost {
			w.WriteHeader(http.StatusMethodNotAllowed)
			return
		}

		var req Request
		if _, err := fmt.Fscanf(r.Body, "%s", &req.Message); err != nil {
			w.WriteHeader(http.StatusBadRequest)
			return
		}

		lastID++
		req.ID = lastID
		requests = append(requests, req)

		w.WriteHeader(http.StatusOK)
	})

	r.HandleFunc("/support/request/photo", func(w http.ResponseWriter, r *http.Request) {
		if r.Method != http.MethodPost {
			w.WriteHeader(http.StatusMethodNotAllowed)
			return
		}

		

		w.WriteHeader(http.StatusOK)
	})

	r.HandleFunc("/support/requests", func(w http.ResponseWriter, r *http.Request) {
		if r.Method != http.MethodGet {
			w.WriteHeader(http.StatusMethodNotAllowed)
			return
		}

		for _, req := range requests {
			fmt.Fprintf(w, "ID: %d, Message: %s\n", req.ID, req.Message)
		}
	})

	log.Fatal(http.ListenAndServe(":8080", r))
}


