<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Sistema de Estoque - Firebase</title>

<style>
/* MESMO CSS QUE VOCÊ TINHA */
body{
    margin:0;
    font-family: Arial, sans-serif;
    background: linear-gradient(135deg,#6a0dad,#00c896,#007bff);
    min-height:100vh;
}
.container{
    background:white;
    margin:20px;
    padding:20px;
    border-radius:30px;
    box-shadow:0 0 15px rgba(0,0,0,0.2);
}
h2,h3{color:#6a0dad;text-align:center;}
.hidden{display:none;}
input, button, select{
    width:100%;
    padding:12px;
    margin-top:10px;
    font-size:16px;
    border-radius:25px;
    border:none;
}
input, select{border:2px solid #6a0dad;}
button{background:#6a0dad;color:white;font-weight:bold;}
button:hover{background:#00c896;}
.products{
    display:grid;
    grid-template-columns:repeat(2,1fr);
    gap:15px;
    max-height:300px;
    overflow-y:auto;
    margin-top:15px;
}
.product{
    border:3px solid #007bff;
    border-radius:25px 5px 25px 5px;
    padding:15px;
    text-align:center;
    cursor:pointer;
    color:#6a0dad;
    font-weight:bold;
}
.product.selected{
    background:#e6fff6;
    border-color:#00c896;
}
table{
    width:100%;
    border-collapse:collapse;
    margin-top:10px;
}
th{
    background:#6a0dad;
    color:white;
}
td,th{
    border:1px solid #ccc;
    padding:8px;
    text-align:center;
}
</style>
</head>
<body>

<!-- MESMAS TELAS -->
<div class="container" id="tela1">
    <h2>Digite seu nome completo</h2>
    <input id="nome" placeholder="Nome completo">
    <button onclick="continuar()">Continuar</button>
</div>

<div class="container hidden" id="tela2">
    <h2>Selecione os produtos</h2>
    <p id="usuario"></p>
    <div class="products" id="listaProdutos"></div>
    <button onclick="retirar()">Retirada</button>
    <button onclick="devolver()">Devolução</button>
    <button onclick="verEstoque()">Estoque</button>
    <button onclick="verMovimentacoes()">Movimentações</button>
</div>

<div class="container hidden" id="tela3">
    <h2>Estoque</h2>
    <table>
        <tr><th>Produto</th><th>Quantidade</th></tr>
        <tbody id="estoque"></tbody>
    </table>
    <button onclick="abrirEdicao()">Editar Estoque</button>
    <button onclick="voltar()">Voltar</button>
</div>

<div class="container hidden" id="telaEditar">
    <h2>Editar Estoque</h2>

    <h3>Alterar quantidade</h3>
    <select id="produtoSelecionado"></select>
    <input type="number" id="novaQtd" placeholder="Nova quantidade">
    <button onclick="salvarEdicao()">Salvar Alteração</button>

    <hr>

    <h3>Adicionar novo produto</h3>
    <input id="novoProdutoNome" placeholder="Nome do produto">
    <input type="number" id="novoProdutoQtd" placeholder="Quantidade inicial">
    <button onclick="adicionarProduto()">Adicionar Produto</button>

    <h3>Remover produto</h3>
    <select id="produtoRemover"></select>
    <button onclick="removerProduto()">Remover Produto</button>

    <button onclick="cancelarEdicao()">Voltar</button>
</div>

<div class="container hidden" id="telaMov">
    <h2>Movimentações</h2>
    <table>
        <tr>
            <th>Usuário</th>
            <th>Produto</th>
            <th>Ação</th>
            <th>Qtd</th>
            <th>Observação</th>
            <th>Data/Hora</th>
        </tr>
        <tbody id="listaMov"></tbody>
    </table>
    <button onclick="voltarMov()">Voltar</button>
</div>

<!-- FIREBASE -->
<script type="module">
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.3.0/firebase-app.js";
import { getDatabase, ref, onValue, set } from "https://www.gstatic.com/firebasejs/10.3.0/firebase-database.js";

window.onload = function(){

// CONFIGURAÇÃO FIREBASE (SUBSTITUA PELOS SEUS DADOS)
const firebaseConfig = {
    apiKey: "SUA_API_KEY",
    authDomain: "SEU_PROJETO.firebaseapp.com",
    databaseURL: "https://SEU_PROJETO.firebaseio.com",
    projectId: "SEU_PROJETO",
    storageBucket: "SEU_PROJETO.appspot.com",
    messagingSenderId: "SENDER_ID",
    appId: "APP_ID"
};

const app = initializeApp(firebaseConfig);
const db = getDatabase(app);

// REFERÊNCIAS FIREBASE
const produtosRef = ref(db, 'produtos');
const movRef = ref(db, 'movimentacoes');

// VARIÁVEIS
let usuario="";
let produtos=[];
let selecionados=[];
let movimentacoes=[];

// SINCRONIZAÇÃO REALTIME COM FIREBASE
onValue(produtosRef, snapshot => {
    produtos = snapshot.val() || [
        {nome:"Mouse",qtd:10},{nome:"Teclado",qtd:8},{nome:"Monitor",qtd:5},
        {nome:"Notebook",qtd:3},{nome:"Headset",qtd:7},{nome:"Headset de treinamento",qtd:7},
        {nome:"Cabo VGA",qtd:15},{nome:"Cabo de Força",qtd:20},{nome:"Fonte Dell",qtd:5},
        {nome:"Fonte HP",qtd:5},{nome:"Fonte Lenovo",qtd:5},{nome:"Desktop Dell",qtd:4},
        {nome:"Desktop HP",qtd:4},{nome:"Desktop Lenovo",qtd:4},{nome:"CPU Dell",qtd:6},
        {nome:"CPU HP",qtd:6},{nome:"CPU Lenovo",qtd:6}
    ]; // fallback caso não exista no DB
    carregarProdutos();
    atualizarTabela();
});

// Movimentações
onValue(movRef, snapshot => {
    movimentacoes = snapshot.val() || [];
});

// FUNÇÕES PARA ATUALIZAR FIREBASE
function salvarProdutosFirebase(){
    set(produtosRef, produtos);
}

function registrarMovFirebase(acao,produto,qtd,obs){
    let data=new Date().toLocaleString();
    movimentacoes.push({usuario,produto,acao,qtd,obs,data});
    set(movRef, movimentacoes);
}

// ---- SOBREESCREVENDO FUNÇÕES ORIGINAIS PARA FIREBASE ----
window.continuar = function(){
    usuario=document.getElementById("nome").value;
    if(usuario==""){
        alert("Digite seu nome!");
        return;
    }
    tela1.classList.add("hidden");
    tela2.classList.remove("hidden");
    document.getElementById("usuario").innerText="Usuário: "+usuario;
    carregarProdutos();
}

function carregarProdutos(){
    let div=document.getElementById("listaProdutos");
    div.innerHTML="";
    produtos.forEach((p,i)=>{
        let d=document.createElement("div");
        d.className="product";
        d.innerText=p.nome;
        d.onclick=()=>selecionar(i,d);
        div.appendChild(d);
    });
}

function selecionar(i,div){
    if(selecionados.includes(i)){
        selecionados=selecionados.filter(x=>x!=i);
        div.classList.remove("selected");
    }else{
        selecionados.push(i);
        div.classList.add("selected");
    }
}

window.retirar = function(){
    if(selecionados.length==0){alert("Selecione um produto"); return;}
    selecionados.forEach(i=>{
        let qtd;
        while(true){
            qtd = prompt(`Quantos ${produtos[i].nome} irá retirar?`);
            if(qtd===null) return;
            if(isNaN(qtd) || qtd.trim()===""){alert("Somente números são aceitos!");}
            else { qtd=parseInt(qtd); break; }
        }
        let obs = prompt(`Observação para ${produtos[i].nome} (opcional):`) || "";
        if(produtos[i].qtd >= qtd){
            produtos[i].qtd -= qtd;
            salvarProdutosFirebase();
            registrarMovFirebase("Retirada",produtos[i].nome,qtd,obs);
        }else{alert("Estoque insuficiente: "+produtos[i].nome);}
    });
    selecionados=[];
    carregarProdutos();
}

window.devolver = function(){
    if(selecionados.length==0){alert("Selecione um produto"); return;}
    selecionados.forEach(i=>{
        let qtd;
        while(true){
            qtd = prompt(`Quantos ${produtos[i].nome} irá devolver?`);
            if(qtd===null) return;
            if(isNaN(qtd) || qtd.trim()===""){alert("Somente números são aceitos!");}
            else { qtd=parseInt(qtd); break; }
        }
        let obs = prompt(`Observação para ${produtos[i].nome} (opcional):`) || "";
        produtos[i].qtd += qtd;
        salvarProdutosFirebase();
        registrarMovFirebase("Devolução",produtos[i].nome,qtd,obs);
    });
    selecionados=[];
    carregarProdutos();
}

window.verEstoque = function(){
    tela2.classList.add("hidden");
    tela3.classList.remove("hidden");
    atualizarTabela();
}

function atualizarTabela(){
    let tbody=document.getElementById("estoque");
    tbody.innerHTML="";
    produtos.forEach(p=>{
        tbody.innerHTML+=`<tr><td>${p.nome}</td><td>${p.qtd}</td></tr>`;
    });
}

window.abrirEdicao = function(){
    tela3.classList.add("hidden");
    telaEditar.classList.remove("hidden");
    let select=document.getElementById("produtoSelecionado");
    let remove=document.getElementById("produtoRemover");
    select.innerHTML=""; remove.innerHTML="";
    produtos.forEach((p,i)=>{
        let opt=document.createElement("option"); opt.value=i; opt.text=p.nome; select.appendChild(opt);
        let opt2=opt.cloneNode(true); remove.appendChild(opt2);
    });
}

window.salvarEdicao = function(){
    let i=document.getElementById("produtoSelecionado").value;
    let qtd=document.getElementById("novaQtd").value;
    produtos[i].qtd=parseInt(qtd);
    salvarProdutosFirebase();
    atualizarTabela();
    cancelarEdicao();
}

window.adicionarProduto = function(){
    let nome=document.getElementById("novoProdutoNome").value;
    let qtd=document.getElementById("novoProdutoQtd").value;
    if(nome==""||qtd==""){alert("Preencha os campos"); return;}
    produtos.push({nome,qtd:parseInt(qtd)});
    salvarProdutosFirebase();
    alert("Produto adicionado!");
    carregarProdutos();
    abrirEdicao();
}

window.removerProduto = function(){
    let i=document.getElementById("produtoRemover").value;
    if(confirm("Deseja remover este produto?")){
        produtos.splice(i,1);
        salvarProdutosFirebase();
        carregarProdutos();
        abrirEdicao();
        atualizarTabela();
    }
}

window.cancelarEdicao = function(){
    telaEditar.classList.add("hidden");
    tela3.classList.remove("hidden");
}

window.verMovimentacoes = function(){
    tela2.classList.add("hidden");
    telaMov.classList.remove("hidden");
    let tb=document.getElementById("listaMov");
    tb.innerHTML="";
    movimentacoes.forEach(m=>{
        tb.innerHTML+=`<tr>
            <td>${m.usuario}</td>
            <td>${m.produto}</td>
            <td>${m.acao}</td>
            <td>${m.qtd}</td>
            <td>${m.obs}</td>
            <td>${m.data}</td>
        </tr>`;
    });
}

window.voltarMov = function(){ telaMov.classList.add("hidden"); tela2.classList.remove("hidden"); }
window.voltar = function(){ tela3.classList.add("hidden"); tela2.classList.remove("hidden"); }

} // fim window.onload
</script>
</body>
</html>
