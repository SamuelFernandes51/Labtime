// ---------------- LOGIN ----------------
async function login() {
    const email = document.getElementById("email").value;
    const senha = document.getElementById("senha").value;

    if (!email || !senha) {
        alert("Preencha todos os campos.");
        return;
    }

    const response = await fetch("http://localhost:3000/auth/login", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ email, senha })
    });

    if (!response.ok) {
        alert("Email ou senha incorretos.");
        return;
    }

    const data = await response.json();

    // SALVA DADOS DO USUÁRIO
    localStorage.setItem("token", data.token);
    localStorage.setItem("nome", data.nome);

    // Vai para o home do professor
    window.location.href = "professor_home.html";
}


// ---------------- CADASTRO ----------------
async function cadastrar() {
    const nome = document.getElementById("nome").value;
    const email = document.getElementById("email").value;
    const senha = document.getElementById("senha").value;

    if (!nome || !email || !senha) {
        alert("Preencha todos os campos!");
        return;
    }

    try {
        const resposta = await fetch("http://localhost:3000/auth/register", {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({ nome, email, senha })
        });

        if (!resposta.ok) {
            const erro = await resposta.text();
            alert("Erro ao cadastrar: " + erro);
            return;
        }

        alert("Cadastro realizado com sucesso!");
        window.location.href = "login.html";

    } catch (err) {
        console.error(err);
        alert("Erro ao conectar ao servidor.");
    }
}
