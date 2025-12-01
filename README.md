# buddingProject
This is my first Git Repository.
<br>
Author - Shivani Shukla
import org.springframework.boot.*;
import org.springframework.boot.autoconfigure.*;
import org.springframework.web.bind.annotation.*;
import org.springframework.stereotype.*;
import org.springframework.http.*;
import java.util.*;
import java.time.*;

// =========================
// MAIN APPLICATION
// =========================
@SpringBootApplication
public class WalletWiseApplication {
    public static void main(String[] args) {
        SpringApplication.run(WalletWiseApplication.class, args);
    }
}

// =========================
// MODELS
// =========================
class Transaction {
    public String category;
    public double amount;
    public String description;
    public String date;

    public Transaction() {}
    public Transaction(String c, double a, String d, String dt){
        category=c; amount=a; description=d; date=dt;
    }
}

// =========================
// CONTROLLER
// =========================
@RestController
public class WalletWiseController {

    private final List<Transaction> txs = new ArrayList<>();

    // Serve the full website
    @GetMapping("/")
    public ResponseEntity<String> home(){
        return ResponseEntity.ok().contentType(MediaType.TEXT_HTML).body(frontendHTML());
    }

    @PostMapping("/add")
    public List<Transaction> add(@RequestBody Map<String,String> body){
        txs.add(new Transaction(
                body.get("category"),
                Double.parseDouble(body.get("amount")),
                body.get("description"),
                LocalDate.now().toString()
        ));
        return txs;
    }

    @GetMapping("/all")
    public List<Transaction> all(){
        return txs;
    }

    @GetMapping("/insights")
    public Map<String,Object> insights(){
        Map<String,Object> out = new HashMap<>();

        if(txs.isEmpty()){
            out.put("message","No transactions yet.");
            return out;
        }

        double total = txs.stream().mapToDouble(t->t.amount).sum();

        Map<String,Double> byCat = new HashMap<>();
        for(Transaction t: txs){
            byCat.put(t.category, byCat.getOrDefault(t.category,0.0)+t.amount);
        }

        String top = byCat.entrySet()
                .stream().max(Map.Entry.comparingByValue())
                .get().getKey();

        List<String> sug = new ArrayList<>();

        if(byCat.getOrDefault("Eating Out",0.0) > total * 0.15)
            sug.add("Eating Out is high. Try 3 home-cooked meals each week.");

        if(byCat.getOrDefault("Entertainment",0.0) > total * 0.10)
            sug.add("Entertainment spending is high. Review subscriptions.");

        if(byCat.getOrDefault("Groceries",0.0) > total * 0.25)
            sug.add("Groceries are large. Try weekly meal plans.");

        if(sug.isEmpty())
            sug.add("Your spending is balanced! Consider auto-saving 10%.");

        out.put("total", total);
        out.put("topCategory", top);
        out.put("byCategory", byCat);
        out.put("suggestions", sug);

        return out;
    }

    // =========================
    // EMBEDDED FRONTEND (HTML + JS)
    // =========================
    public String frontendHTML(){
        return """
<!DOCTYPE html>
<html>
<head>
<title>WalletWise AI</title>
<style>
body { font-family: Arial; background:#f5f6ff; margin:0; padding:20px; }
.container { max-width:700px; margin:auto; }
.card { background:white; padding:20px; margin-top:15px; border-radius:8px; box-shadow:0 2px 5px rgba(0,0,0,0.1);}
button { padding:10px 16px; background:#0b5cff; color:white; border:none; border-radius:6px; cursor:pointer;}
table { width:100%; border-collapse:collapse; margin-top:10px;}
td,th { padding:8px; border-bottom:1px solid #ddd; text-align:left;}
</style>
</head>

<body>
<div class="container">
<h1>WalletWise AI: Spend Smart, Save Smarter</h1>

<div class="card">
<h3>Add Transaction</h3>
<label>Category:</label>
<select id="cat">
  <option>Groceries</option>
  <option>Rent</option>
  <option>Eating Out</option>
  <option>Entertainment</option>
  <option>Transport</option>
  <option>Others</option>
</select><br><br>

<label>Amount:</label>
<input id="amt" type="number"><br><br>

<label>Description:</label>
<input id="desc" type="text"><br><br>

<button onclick="addTx()">Add</button>
</div>

<div class="card">
<h3>Transactions</h3>
<table id="tbl"></table>
</div>

<div class="card">
<h3>AI Insights</h3>
<div id="ins"></div>
</div>
</div>

<script>
async function addTx(){
    const body = {
        category: document.getElementById('cat').value,
        amount: document.getElementById('amt').value,
        description: document.getElementById('desc').value
    };

    await fetch('/add',{method:'POST', headers:{'Content-Type':'application/json'}, body:JSON.stringify(body)});
    load();
}

async function load(){
    const tx = await (await fetch('/all')).json();
    const ins = await (await fetch('/insights')).json();

    // fill table
    let html = "<tr><th>Date</th><th>Category</th><th>Description</th><th>Amount</th></tr>";
    tx.forEach(t => {
        html += `<tr><td>${t.date}</td><td>${t.category}</td><td>${t.description}</td><td>${t.amount}</td></tr>`;
    });
    document.getElementById("tbl").innerHTML = html;

    // insights
    let ihtml = "";
    if(ins.message){
        ihtml = "<p>No transactions yet.</p>";
    } else {
        ihtml += `<p><b>Total:</b> ${ins.total}</p>`;
        ihtml += `<p><b>Top Category:</b> ${ins.topCategory}</p>`;
        ihtml += "<b>Suggestions:</b><ul>";
        ins.suggestions.forEach(s => ihtml += `<li>${s}</li>`);
        ihtml += "</ul>";
    }
    document.getElementById("ins").innerHTML = ihtml;
}

load();
</script>
</body>
</html>
""";
    }
}
