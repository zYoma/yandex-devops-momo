FROM golang:alpine
WORKDIR /build
COPY . .
RUN go build -o backend ./cmd/api
CMD ["./backend"]