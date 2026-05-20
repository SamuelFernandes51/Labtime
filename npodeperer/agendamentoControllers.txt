const pool = require("../db");

module.exports = {
    // =====================================================
    // 📌 1) Criar agendamento (uma aula)
    // =====================================================
    async criar(req, res) {

        try {
            // Adicionado estado, mas vamos usar um valor padrão na query
            const { professor_id, espaco_id, data, numero_aula } = req.body;

            if (!professor_id || !espaco_id || !data || !numero_aula) {
                return res.status(400).json({ erro: "Dados incompletos." });
            }

            // CORREÇÃO FINAL: Usamos 'indisponivel' conforme a restrição agendamentos_estado_check.
            const query = `
                INSERT INTO agendamentos (professor_id, espaco_id, data, numero_aula, estado)
                VALUES ($1, $2, $3, $4, 'indisponivel')
                RETURNING *;
            `;

            const values = [professor_id, espaco_id, data, numero_aula];

            const result = await pool.query(query, values);

            return res.status(201).json({
                mensagem: "Agendamento criado com sucesso.",
                agendamento: result.rows[0]
            });

        } catch (erro) {
            // Se for erro de UNIQUE (conflito)
            if (erro.code === "23505") {
                return res.status(400).json({
                    erro: "Este laboratório já está ocupado nesse dia e horário."
                });
            }
            // Se for erro de Restrição de Verificação (CHECK CONSTRAINT) 23514
            if (erro.code === "23514") {
                return res.status(400).json({
                    erro: "O estado do agendamento ('indisponivel') não é válido. Verifique a restrição de valores permitidos no banco de dados."
                });
            }
            
            console.error(erro);
            return res.status(500).json({ erro: "Erro ao criar agendamento." });
        }
    },


    // =====================================================
    // 📌 2) Criar vários agendamentos de uma vez
    // =====================================================
    async criarMultiplas(req, res) {
        try {
            const { professor_id, espaco_id, data, aulas } = req.body;

            if (!professor_id || !espaco_id || !data || !aulas || aulas.length === 0) {
                return res.status(400).json({ erro: "Dados incompletos." });
            }

            const inseridos = [];
            const erros = [];

            for (let numero_aula of aulas) {
                try {
                    // CORREÇÃO FINAL: Usamos 'indisponivel' conforme a restrição agendamentos_estado_check.
                    const query = `
                        INSERT INTO agendamentos (professor_id, espaco_id, data, numero_aula, estado)
                        VALUES ($1, $2, $3, $4, 'indisponivel')
                        RETURNING *;
                    `;

                    const result = await pool.query(query, [
                        professor_id, espaco_id, data, numero_aula
                    ]);

                    inseridos.push(result.rows[0]);

                } catch (err) {
                    if (err.code === "23505") {
                        erros.push(`Conflito na aula ${numero_aula}`);
                    } else {
                        erros.push(`Erro desconhecido na aula ${numero_aula}`);
                    }
                }
            }

            return res.status(207).json({
                mensagem: "Processo finalizado",
                inseridos,
                erros
            });

        } catch (erro) {
            console.error(erro);
            return res.status(500).json({ erro: "Erro ao criar múltiplos agendamentos." });
        }
    },


    // =====================================================
    // 📌 3) Deletar agendamento - AGORA COM VERIFICAÇÃO DE PROPRIEDADE
    // =====================================================
    async deletar(req, res) {
        try {
            const { id } = req.params;
            // ID do professor logado injetado pelo middleware 'verificarToken'
            const professorLogadoId = req.professorId; 

            // 1. Verificação de ID de professor logado (Garantia do middleware)
            if (!professorLogadoId) {
                // Se o middleware falhou (improvável se o token for verificado), retorna 401.
                return res.status(401).json({ erro: "Professor logado não identificado. Acesso negado." });
            }

            // 2. BUSCAR o ID do professor que realmente criou este agendamento
            const checkQuery = `
                SELECT professor_id FROM agendamentos WHERE id = $1;
            `;
            const checkResult = await pool.query(checkQuery, [id]);

            if (checkResult.rowCount === 0) {
                return res.status(404).json({ erro: "Agendamento não encontrado." });
            }

            const agendamento = checkResult.rows[0];

            // 3. COMPARAR: Se o dono do agendamento (do banco) é o professor logado (do token)
            // Usamos String() para garantir comparação entre tipos iguais (ex: number vs string)
            if (String(agendamento.professor_id) !== String(professorLogadoId)) {
                // Retorna 403 Forbidden se o professor logado não for o dono
                return res.status(403).json({ erro: "Ação não permitida. Você só pode excluir seus próprios agendamentos." });
            }

            // 4. Se a verificação passar, executa o DELETE
            // Usamos o professor_id logado na query para redundância e segurança extra
            const deleteQuery = `DELETE FROM agendamentos WHERE id = $1 AND professor_id = $2`;
            const result = await pool.query(deleteQuery, [id, professorLogadoId]);


            if (result.rowCount === 0) {
                return res.status(404).json({ erro: "Agendamento não encontrado ou não pertence a você." });
            }

            return res.json({ mensagem: "Agendamento removido com sucesso." });

        } catch (erro) {
            console.error(erro);
            return res.status(500).json({ erro: "Erro ao deletar agendamento." });
        }
    },


    // =====================================================
    // 📌 4) Listar agendamentos (Todos os professores e espaços) - CORRIGIDA (Versão Limpa)
    // =====================================================
    async listar(req, res) {
        try {
            // CORREÇÃO: Adicionamos a.professor_id, p.nome, e.nome e a.estado na SELECT
            const result = await pool.query(`
                SELECT 
                    a.id, 
                    a.professor_id, /* ESSENCIAL PARA O FRONTEND */
                    p.nome AS professor, 
                    e.nome AS laboratorio, 
                    TO_CHAR(a.data, 'DD/MM/YYYY') AS data, 
                    a.numero_aula,
                    a.estado
                FROM agendamentos a
                JOIN professores p ON p.id = a.professor_id
                JOIN espacos e ON e.id = a.espaco_id
                ORDER BY a.data ASC, a.numero_aula ASC;
            `);

            return res.json(result.rows);

        } catch (erro) {
            console.error("ERRO COMPLETO NA ROTA LISTAR:", erro); // Log de erro mais detalhado
            return res.status(500).json({ erro: "Erro ao listar agendamentos. Verifique o log do servidor para detalhes." });
        }
    }
};