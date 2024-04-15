package main

import (
	"fmt"
	"log"
	"net/http"

	"github.com/gin-gonic/gin"
)

type Request struct {
	ID       string `json:"id"`
	Message  string `json:"message"`
	Severity string `json:"severity"`
}

var requests []Request

func main() {
	r := gin.Default()


	r.POST("/support/request", func(c *gin.Context) {
		var req Request
		if err := c.BindJSON(&req); err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}
		req.ID = fmt.Sprintf("REQ%d", len(requests)+1)
		requests = append(requests, req)
		c.JSON(http.StatusOK, req)
	})

	
	r.POST("/support/request/:id/photo", func(c *gin.Context) {
		id := c.Param("id")
		// Lógica para receber e salvar a foto
		c.JSON(http.StatusOK, gin.H{"message": "Photo uploaded successfully"})
	})


	r.PUT("/support/request/:id/classify", func(c *gin.Context) {
		id := c.Param("id")
		severity := c.Query("severity")
		// Lógica para classificar a solicitação com base na gravidade
		for i, req := range requests {
			if req.ID == id {
				requests[i].Severity = severity
				break
			}
		}
		c.JSON(http.StatusOK, gin.H{"message": "Request classified successfully"})
	})


	r.GET("/support/requests", func(c *gin.Context) {
		c.JSON(http.StatusOK, requests)
	})

	if err := r.Run(":8080"); err != nil {
		log.Fatal(err)
	}
}

