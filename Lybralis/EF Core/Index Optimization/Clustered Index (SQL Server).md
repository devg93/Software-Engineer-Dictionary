![[Pasted image 20251025223105.png]]- **ფიზიკურად** ალაგებს მონაცემებს დისკზე
- **მხოლოდ ერთი** Clustered Index შეიძლება ცხრილზე
- თავად **ცხრილია** (data-ა და index ერთია)

**როდის გამოიყენო:**

- ✅ Primary Key-ზე (default)
- ✅ თუ ხშირად ეძებ **range queries** (BETWEEN, >, <)
- ✅ თუ ხშირად აკეთებ **ORDER BY**
- ![[Pasted image 20251025224510.png]]