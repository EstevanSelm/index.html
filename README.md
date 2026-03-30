# index.html

/**
 * ============================================================
 * api.js — Camada de Integração Front-end / Back-end
 * InclusãoTech | PIM V | UNIP 2026
 * ============================================================
 * Responsável back-end: [seu nome]
 * Repositório: https://dinizzs2.github.io/PIM---Site-Inclusivo/
 *
 * ARQUITETURA:
 * Como o GitHub Pages só serve arquivos estáticos, este arquivo
 * replica em JavaScript a lógica de negócio modelada nas classes
 * C++ do back-end (Evento, Participante, Inscricao, Certificado).
 * A persistência usa localStorage no lugar de fstream/CSV.
 *
 * CLASSES C++ ESPELHADAS:
 *   Evento        → DB.evento
 *   Atividade     → DB.atividades[]
 *   Participante  → DB.participantes[]
 *   Inscricao     → DB.inscricoes[]
 *   Certificado   → gerado em criarInscricao()
 *
 * EXCEÇÕES C++ ESPELHADAS:
 *   VagasEsgotadasException    → throw { tipo: "VagasEsgotadas" }
 *   InscricaoDuplicadaException→ throw { tipo: "InscricaoDuplicada" }
 *   DadosInvalidosException    → throw { tipo: "DadosInvalidos" }
 * ============================================================
 */
 
(function () {
  "use strict";
 
  // ----------------------------------------------------------
  // SEÇÃO 1 — BANCO DE DADOS EM MEMÓRIA
  // Espelha as coleções STL do back-end C++:
  //   vector<Atividade>    → DB.atividades
  //   map<string,bool>     → DB.inscritos  (anti-duplicata)
  //   vector<Participante> → DB.participantes
  //   vector<Inscricao>    → DB.inscricoes
  // ----------------------------------------------------------
 
  // Carrega estado salvo anteriormente (persistência entre sessões)
  function _carregar(chave, padrao) {
    try {
      var raw = localStorage.getItem("inclusaotech_" + chave);
      return raw ? JSON.parse(raw) : padrao;
    } catch (e) {
      return padrao;
    }
  }
 
  // Salva estado (espelha fstream / Evento::salvarEmArquivo())
  function _salvar(chave, valor) {
    try {
      localStorage.setItem("inclusaotech_" + chave, JSON.stringify(valor));
    } catch (e) {
      console.warn("[API] Falha ao salvar:", chave, e);
    }
  }
 
  var DB = {
    // ── Evento principal (Evento::Evento(...)) ──────────────
    evento: {
      id: 1,
      titulo: "Semana Acadêmica de TI 2026",
      descricao: "O maior evento acadêmico de TI na UNIP, totalmente acessível.",
      dataInicio: "15/05/2026",
      dataFim: "18/05/2026",
      vagasTotal: 200,
      vagasDisponiveis: _carregar("vagas", 200),
      modalidade: "Híbrido",
      acessivelLibras: true
    },
 
    // ── Atividades (vector<Atividade>) ──────────────────────
    // Mapeadas da programacao.html (dados que estavam hardcoded)
    atividades: [
      {
        id: 1,
        titulo: "Abertura: O Futuro da TI Inclusiva",
        horarioInicio: "09:00",
        horarioFim: "10:30",
        palestrante: "Dr. Silva e Maria Souza",
        descricao: "Uma visão geral sobre como o mercado de tecnologia está se adaptando para receber profissionais surdos e neurodivergentes.",
        cargaHoraria: 1.5,
        recursos: ["Libras", "Audiodescrição"]
      },
      {
        id: 2,
        titulo: "Desenvolvimento Web e WCAG",
        horarioInicio: "11:00",
        horarioFim: "12:30",
        palestrante: "Ana Paula (Dev Sênior)",
        descricao: "Oficina prática sobre como aplicar as diretrizes de acessibilidade (WCAG) em projetos reais de front-end.",
        cargaHoraria: 1.5,
        recursos: ["Libras"]
      },
      {
        id: 3,
        titulo: "Arquitetura de Back-end em C++",
        horarioInicio: "14:00",
        horarioFim: "16:00",
        palestrante: "Carlos Mendes",
        descricao: "Explorando orientação a objetos e boas práticas na construção de sistemas robustos usando C++.",
        cargaHoraria: 2.0,
        recursos: ["Libras", "Material em Braille"]
      },
      {
        id: 4,
        titulo: "Mesa Redonda: Liderança e Negociação",
        horarioInicio: "16:30",
        horarioFim: "18:00",
        palestrante: "Mediação: Prof. Roberto",
        descricao: "Debate sobre soft skills, comunicação organizacional e como gerenciar equipes diversas em projetos de TI.",
        cargaHoraria: 1.5,
        recursos: ["Libras"]
      }
    ],
 
    // ── Coleções carregadas do localStorage ─────────────────
    participantes: _carregar("participantes", []),
    inscricoes:    _carregar("inscricoes", []),
    inscritos:     _carregar("inscritos", {}),  // map<email_eventoId, bool>
 
    // Contadores de IDs
    nextId: _carregar("nextId", { part: 1, insc: 1, cert: 1 })
  };
 
 
  // ----------------------------------------------------------
  // SEÇÃO 2 — GERAÇÃO DE HASH DE CERTIFICADO
  // Espelha: Certificado::gerarHash(partId, evId) do C++
  // ----------------------------------------------------------
  function _gerarHash(partId, evId) {
    return "CERT-" + evId + "-" + partId + "-" + (partId * evId * 7919);
  }
 
 
  // ----------------------------------------------------------
  // SEÇÃO 3 — FUNÇÕES PÚBLICAS DA API
  // Cada função espelha um método ou construtor C++
  // ----------------------------------------------------------
 
  /**
   * getEvento()
   * Espelha: Evento::getTitulo(), getVagasDisponiveis(), etc.
   * Usado em: index.html (stats do hero)
   */
  function getEvento() {
    var totalCH = DB.atividades.reduce(function (s, a) {
      return s + a.cargaHoraria;
    }, 0);
    return Object.assign({}, DB.evento, {
      totalAtividades: DB.atividades.length,
      cargaHorariaTotal: totalCH
    });
  }
 
  /**
   * getAtividades()
   * Espelha: Evento::getAtividades() → vector<Atividade>
   * Usado em: programacao.html
   */
  function getAtividades() {
    return DB.atividades;
  }
 
  /**
   * criarInscricao(dados)
   * Espelha: new Inscricao(id, participante, evento, inscritos)
   *
   * Lança (try/catch no formulario.html):
   *   { tipo: "DadosInvalidos",      campo: "nome"|"email"|"curso" }
   *   { tipo: "VagasEsgotadas",      evento: string }
   *   { tipo: "InscricaoDuplicada",  email: string }
   *
   * @param {Object} dados - { nome, email, telefone, curso, precisaAcessibilidade }
   * @returns {Object} - { ok, participante, inscricao, certificado }
   */
  function criarInscricao(dados) {
 
    // ── Validação — DadosInvalidosException ────────────────
    if (!dados.nome || dados.nome.trim() === "")
      throw { tipo: "DadosInvalidos", campo: "nome" };
    if (!dados.email || dados.email.indexOf("@") < 0)
      throw { tipo: "DadosInvalidos", campo: "email" };
    if (!dados.curso || dados.curso === "")
      throw { tipo: "DadosInvalidos", campo: "curso" };
 
    // ── Vagas — VagasEsgotadasException ────────────────────
    if (DB.evento.vagasDisponiveis <= 0)
      throw { tipo: "VagasEsgotadas", evento: DB.evento.titulo };
 
    // ── Duplicata — InscricaoDuplicadaException ─────────────
    // Usa email como chave (no lugar do CPF que não está no form)
    var chave = dados.email.toLowerCase() + "_" + DB.evento.id;
    if (DB.inscritos[chave])
      throw { tipo: "InscricaoDuplicada", email: dados.email };
 
    // ── Cria Participante (Participante::Participante(...)) ──
    var part = {
      id: DB.nextId.part++,
      nome: dados.nome.trim(),
      email: dados.email.trim().toLowerCase(),
      telefone: dados.telefone || "",
      curso: dados.curso,
      precisaAcessibilidade: dados.precisaAcessibilidade || false,
      tipoAcessibilidade: dados.precisaAcessibilidade ? "Libras" : "Nenhuma"
    };
    DB.participantes.push(part);
 
    // ── Cria Inscrição (Inscricao::Inscricao(...)) ───────────
    var hoje = new Date().toLocaleDateString("pt-BR");
    var insc = {
      id: DB.nextId.insc++,
      participanteId: part.id,
      eventoId: DB.evento.id,
      dataInscricao: hoje,
      status: "PENDENTE"    // Inscricao::Status::PENDENTE
    };
    DB.inscricoes.push(insc);
    DB.inscritos[chave] = true;
    DB.evento.vagasDisponiveis--;
 
    // ── Confirma (Inscricao::confirmar()) ───────────────────
    insc.status = "CONFIRMADA";
 
    // ── Gera Certificado (Certificado::emitir()) ────────────
    var totalCH = DB.atividades.reduce(function (s, a) {
      return s + a.cargaHoraria;
    }, 0);
    var cert = {
      id: DB.nextId.cert++,
      participanteId: part.id,
      eventoId: DB.evento.id,
      cargaHoraria: totalCH,
      dataEmissao: hoje,
      hash: _gerarHash(part.id, DB.evento.id)
    };
 
    // ── Persiste tudo (fstream / salvarEmArquivo()) ─────────
    _salvar("participantes", DB.participantes);
    _salvar("inscricoes", DB.inscricoes);
    _salvar("inscritos", DB.inscritos);
    _salvar("vagas", DB.evento.vagasDisponiveis);
    _salvar("nextId", DB.nextId);
 
    // Salva participante atual para area-participante.html
    _salvar("participanteAtual", {
      participante: part,
      inscricao: insc,
      certificado: cert,
      atividades: DB.atividades   // inscrições nas atividades
    });
 
    return { ok: true, participante: part, inscricao: insc, certificado: cert };
  }
 
  /**
   * getParticipanteAtual()
   * Espelha: Participante logado + Inscricao + Certificado
   * Usado em: area-participante.html
   * @returns {Object|null}
   */
  function getParticipanteAtual() {
    return _carregar("participanteAtual", null);
  }
 
  /**
   * logout()
   * Limpa sessão do participante (não apaga inscrições)
   */
  function logout() {
    localStorage.removeItem("inclusaotech_participanteAtual");
  }
 
  /**
   * resetarTudo()
   * Útil durante testes — limpa todo o estado salvo
   */
  function resetarTudo() {
    var chaves = ["participantes","inscricoes","inscritos","vagas","nextId","participanteAtual"];
    chaves.forEach(function (c) {
      localStorage.removeItem("inclusaotech_" + c);
    });
    console.log("[API] Estado resetado. Recarregue a página.");
  }
 
 
  // ----------------------------------------------------------
  // SEÇÃO 4 — EXPORTAÇÃO
  // Expõe window.API para todas as páginas do site
  // ----------------------------------------------------------
  window.API = {
    getEvento:            getEvento,
    getAtividades:        getAtividades,
    criarInscricao:       criarInscricao,
    getParticipanteAtual: getParticipanteAtual,
    logout:               logout,
    resetarTudo:          resetarTudo
  };
 
  console.log("[API] InclusãoTech carregada. Vagas disponíveis:", DB.evento.vagasDisponiveis);
 
})();
