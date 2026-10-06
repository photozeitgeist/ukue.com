---
title: "An Order and Its Receipt Email in One SQLite Transaction With ukue's EnqueueTx"
date: 2026-10-06
draft: false
featured_image: "https://ukue.com/images/usecase-order-transaction.png"
tags: ["use cases", "go", "sqlite"]
---

Take a small online shop that runs as a single Go program with its data in SQLite: one file to back up and no database server to look after. When an order comes in, the shop saves it, and a receipt email goes out in the background.

The risk is in the gap between those two steps. If the order is saved and the program crashes before the email job is queued, the customer pays and never gets a receipt. If the job is queued first and saving the order then fails, the customer gets a receipt for an order that doesn't exist. With a separate queue server, closing that gap takes real work, usually an "outbox" table in the database and a second process that copies its rows into the queue. With ukue, the queue can live inside the shop's own database, so the order and its job go into the same transaction, and either both are saved or neither is.

![Inside one SQLite transaction, the shop inserts order 1042 and adds its receipt job with EnqueueTx. On commit both rows are saved in shop.db and a worker sends the receipt. On an error the rollback removes both.](https://ukue.com/images/usecase-order-transaction.png)

## The Queue Inside the Shop's Database

`ukue.OpenDB` takes the shop's existing database handle and adds ukue's two tables, `ukue_jobs` and `ukue_meta`, next to the shop's own:

```go
db, err := sql.Open("sqlite3", "file:shop.db?_busy_timeout=5000&_synchronous=FULL")
if err != nil {
	log.Fatal(err)
}
q, err := ukue.OpenDB(db)
if err != nil {
	log.Fatal(err)
}
```

When it adds its tables, ukue also switches the file to SQLite's WAL mode, which most SQLite web apps use anyway. The connection string matters for one reason. ukue runs its own connections with `synchronous = FULL`, but a job added inside the shop's transaction is committed on the shop's connection, so that connection needs the same setting for the job to survive a power cut. With the go-sqlite3 driver, `_synchronous=FULL` in the connection string does it.

## Saving the Order

```go
func placeOrder(ctx context.Context, customerID, totalCents int64) error {
	tx, err := db.BeginTx(ctx, nil)
	if err != nil {
		return err
	}
	defer tx.Rollback()

	res, err := tx.ExecContext(ctx,
		`INSERT INTO orders (customer_id, total_cents) VALUES (?, ?)`, customerID, totalCents)
	if err != nil {
		return err
	}
	orderID, err := res.LastInsertId()
	if err != nil {
		return err
	}
	payload := []byte(strconv.FormatInt(orderID, 10))
	if _, err := q.EnqueueTx(ctx, tx, "receipts", payload); err != nil {
		return err
	}
	return tx.Commit()
}
```

`EnqueueTx` writes the job's row inside the shop's transaction. If anything fails before `Commit`, the deferred `Rollback` removes the order and the job together. If the commit goes through, both exist, and workers pick the job up at their next look, within a second with the default settings.

## Sending the Receipt

The same program runs the worker in a goroutine:

```go
go q.Work(ctx, "receipts", func(ctx context.Context, job *ukue.Job) error {
	orderID, err := strconv.ParseInt(string(job.Payload), 10, 64)
	if err != nil {
		return ukue.Permanent(err) // not an order number: retrying won't help
	}
	return sendReceipt(ctx, orderID)
})
```

The job carries only the order number. `sendReceipt` loads the order when it runs, so the email shows the order as it stands when the email is sent. If the mail service is down, `sendReceipt` returns an error, and ukue tries again after 10 seconds, then 20, then 40, doubling up to an hour.

## One File, One Backup

Because the jobs live in the shop's database, one backup of `shop.db`, taken with SQLite's own backup command as for any database in WAL mode, holds the orders and their jobs as of the same moment. Restore it, and every order whose receipt hadn't gone out yet still has its job waiting.

## What It Doesn't Change

The job is saved exactly when the order is, but it still runs at least once. If the program dies after the email went out and before the job was marked done, the receipt goes out again after the restart. A second receipt is harmless. A job where a repeat would cost money, like charging a card, should pass the order number to the payment provider as an idempotency key, so that the provider recognises a repeat.
