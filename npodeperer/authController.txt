const pool = require("../db");
const bcrypt = require("bcrypt");
const jwt = require("jsonwebtoken");

module.exports = {
    async register(req, res) {
        const { nome, email, senha } = req.body;

        if (!nome || !email || !senha) {
            return res.status(400).send("Dados incompletos.");
        }

        try {
            const senhaHash = await bcrypt.hash(senha, 10);

            await pool.query(
                "INSERT INTO professores (nome, email, senha_hash) VALUES ($1, $2, $3)",
                [nome, email, senhaHash]
            );

            res.status(201).send("Professor criado");
        } catch (err) {
            console.error(err);
            res.status(500).send("Erro ao cadastrar professor.");
        }
    },


    async login(req, res) {
        const { email, senha } = req.body;

        try {
            const result = await pool.query(
                "SELECT id, nome, senha_hash FROM professores WHERE email = $1", // Selecionando apenas os campos necessários
                [email]
            );

            if (result.rows.length === 0) {
                return res.status(401).send("Credenciais inválidas.");
            }

            const usuario = result.rows[0];

            const senhaCorreta = await bcrypt.compare(senha, usuario.senha_hash);
            if (!senhaCorreta) {
                return res.status(401).send("Credenciais inválidas.");
            }

            const token = jwt.sign(
                { id: usuario.id, nome: usuario.nome },
                process.env.JWT_SECRET,
                { expiresIn: "8h" }
            );

            // CORREÇÃO: Enviar o nome e o id na resposta JSON para que o frontend (auth.js)
            // possa salvá-los no localStorage.
            res.json({ 
                token: token,
                nome: usuario.nome, 
                id: usuario.id 
            });
        } catch (err) {
            console.error(err);
            res.status(500).send("Erro no servidor.");
        }
    }
};