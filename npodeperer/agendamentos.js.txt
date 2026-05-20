const express = require('express');
const router = express.Router();
const agendamentoController = require('../controllers/agendamentoController');
const jwt = require('jsonwebtoken'); // 1. Importar a biblioteca JWT

// ⚠️ SUA CHAVE SECRETA: Deve ser a mesma chave usada para assinar o token JWT.
// É altamente recomendado armazenar isso em variáveis de ambiente (process.env.JWT_SECRET).
const JWT_SECRET = 'segredo123'; 

// =================================================================
// ✅ MIDDLEWARE REAL DE AUTENTICAÇÃO JWT
// =================================================================
const verificarToken = (req, res, next) => {
    // 1. Tenta obter o cabeçalho de autorização
    const authHeader = req.headers.authorization;

    if (!authHeader) {
        return res.status(401).json({ erro: "Acesso negado. Token não fornecido." });
    }

    // O cabeçalho vem como "Bearer <token>"
    const token = authHeader.split(' ')[1];

    if (!token) {
        return res.status(401).json({ erro: "Token mal formatado." });
    }

    try {
        // 2. Verifica e decodifica o token
        const decoded = jwt.verify(token, JWT_SECRET);
        
        // 3. O payload do token decodificado contém o ID do professor (ou 'sub', dependendo de como você o nomeou)
        // ** Assumimos que o ID está no campo 'id' do payload **
        req.professorId = decoded.id; 

        next(); // Se tudo OK, continua para o controlador
    } catch (err) {
        // Trata tokens inválidos (expirados, adulterados, chave secreta incorreta)
        console.error("Erro na verificação do token:", err.message);
        return res.status(401).json({ erro: "Token inválido ou expirado." });
    }
};


// 4) Rota para listar todos os agendamentos (GET /agendamentos)
router.get('/', agendamentoController.listar);

// 1) Rota para criar um único agendamento (POST /agendamentos)
router.post('/', agendamentoController.criar);

// 2) Rota para criar múltiplos agendamentos (POST /agendamentos/multiplas)
router.post('/multiplas', agendamentoController.criarMultiplas);

// 3) Rota para deletar um agendamento por ID (DELETE /agendamentos/:id)
// ✅ APLICAÇÃO DO MIDDLEWARE REAL
router.delete('/:id', verificarToken, agendamentoController.deletar);

module.exports = router;