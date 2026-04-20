import React, { useState, useEffect, useRef, useCallback } from "react";
import { base44 } from "@/api/base44Client";
import CRTScreen from "../components/terminal/CRTScreen";
import TerminalLine from "../components/terminal/TerminalLine";
import TerminalInput from "../components/terminal/TerminalInput";
import LoadingBar from "../components/terminal/LoadingBar";
import useTypewriter from "../hooks/useTypewriter";
import { normalizeInput, getPhaseText } from "../lib/terminalUtils";

const SESSION_ID = "main";

export default function Terminal() {
  const [faseAtual, setFaseAtual] = useState(1);
  const [acertos, setAcertos] = useState(0);
  const [erros, setErros] = useState(0);
  const [screenLines, setScreenLines] = useState([]);
  const [inputValue, setInputValue] = useState("");
  const [inputDisabled, setInputDisabled] = useState(true);
  const [showLoadingBar, setShowLoadingBar] = useState(false);
  const [loadingDuration, setLoadingDuration] = useState(60);
  const [showFinalMessage, setShowFinalMessage] = useState(false);
  const [phaseText, setPhaseText] = useState("");
  const [stateId, setStateId] = useState(null);
  const scrollRef = useRef(null);

  // Refs para ter sempre o valor atual nas closures
  const stateIdRef = useRef(null);
  const screenLinesRef = useRef([]);

  const { displayedText, isComplete } = useTypewriter(phaseText, 25);

  // Scroll to bottom when content changes
  useEffect(() => {
    if (scrollRef.current) {
      scrollRef.current.scrollTop = scrollRef.current.scrollHeight;
    }
  }, [displayedText, screenLines, showLoadingBar]);

  // Enable input when typewriter completes
  useEffect(() => {
    if (isComplete && phaseText && !showLoadingBar && !showFinalMessage) {
      setInputDisabled(false);
    }
  }, [isComplete, phaseText, showLoadingBar, showFinalMessage]);

  // Keep refs in sync
  useEffect(() => {
    stateIdRef.current = stateId;
  }, [stateId]);

  useEffect(() => {
    screenLinesRef.current = screenLines;
  }, [screenLines]);

  // Push full state to entity
  const pushState = useCallback(async (overrides = {}) => {
    const data = {
      session_id: SESSION_ID,
      fase_atual: overrides.fase_atual ?? faseAtual,
      acertos: overrides.acertos ?? acertos,
      erros: overrides.erros ?? erros,
      screen_lines: overrides.screen_lines ?? screenLinesRef.current,
      phase_text: overrides.phase_text ?? phaseText,
      current_input: overrides.current_input ?? inputValue,
      is_loading_bar: overrides.is_loading_bar ?? false,
      loading_duration: overrides.loading_duration ?? loadingDuration,
      show_final_message: overrides.show_final_message ?? false,
    };

    const id = stateIdRef.current;
    if (id) {
      await base44.entities.TerminalState.update(id, data);
    } else {
      const existing = await base44.entities.TerminalState.filter({ session_id: SESSION_ID });
      if (existing.length > 0) {
        stateIdRef.current = existing[0].id;
        setStateId(existing[0].id);
        await base44.entities.TerminalState.update(existing[0].id, data);
      } else {
        const created = await base44.entities.TerminalState.create(data);
        stateIdRef.current = created.id;
        setStateId(created.id);
      }
    }
  }, [faseAtual, acertos, erros, phaseText, inputValue, loadingDuration]);

  // Initialize
  useEffect(() => {
    async function init() {
      const existing = await base44.entities.TerminalState.filter({ session_id: SESSION_ID });
      if (existing.length > 0) {
        stateIdRef.current = existing[0].id;
        setStateId(existing[0].id);
      } else {
        const created = await base44.entities.TerminalState.create({ session_id: SESSION_ID });
        stateIdRef.current = created.id;
        setStateId(created.id);
      }
      doStartPhase(1, [], 0, 0);
    }
    init();
  }, []);

  // Sync input changes in real time
  useEffect(() => {
    if (!stateIdRef.current) return;
    const timeout = setTimeout(() => {
      base44.entities.TerminalState.update(stateIdRef.current, {
        current_input: inputValue,
      }).catch(() => {});
    }, 150);
    return () => clearTimeout(timeout);
  }, [inputValue]);

  // ----------------------------------------------------------------
  // Phase management
  // ----------------------------------------------------------------
  function doStartPhase(fase, lines, hits, misses) {
    setInputDisabled(true);
    setInputValue("");
    setShowLoadingBar(false);
    setShowFinalMessage(false);
    setFaseAtual(fase);

    if (fase === 8) {
      doPhase8(hits, misses);
      return;
    }

    const text = getPhaseText(fase);
    setScreenLines(lines);
    screenLinesRef.current = lines;
    setPhaseText("");

    setTimeout(() => {
      setPhaseText(text);
      // Sync to entity
      const id = stateIdRef.current;
      if (id) {
        base44.entities.TerminalState.update(id, {
          fase_atual: fase,
          acertos: hits,
          erros: misses,
          screen_lines: lines,
          phase_text: text,
          current_input: "",
          is_loading_bar: false,
          show_final_message: false,
        }).catch(() => {});
      }
    }, 100);
  }

  function addLine(text, currentLines) {
    const newLines = [...currentLines, text];
    setScreenLines(newLines);
    screenLinesRef.current = newLines;
    return newLines;
  }

  function handleSubmit() {
    if (inputDisabled) return;
    const raw = inputValue;
    const normalized = normalizeInput(raw);
    setInputValue("");

    const newLines = addLine(`> ${raw}`, screenLinesRef.current);

    // Sync the typed line immediately
    if (stateIdRef.current) {
      base44.entities.TerminalState.update(stateIdRef.current, {
        screen_lines: newLines,
        current_input: "",
      }).catch(() => {});
    }

    processInput(normalized, newLines);
  }

  function processInput(normalized, currentLines) {
    switch (faseAtual) {
      case 1: handleFase1(normalized, currentLines); break;
      case 2: handleFase2(normalized, currentLines); break;
      case 3: handleFase3(normalized, currentLines); break;
      case 4: handleFase4(normalized, currentLines); break;
      case 5: handleFase5(normalized, currentLines); break;
      case 6: handleFase6(normalized, currentLines); break;
      case 7: handleFase7(normalized, currentLines); break;
      default: break;
    }
  }

  function handleFase1(input, lines) {
    if (input === "SIM") {
      const newLines = addLine("Luz da Sala Ligada", lines);
      syncLines(newLines);
      setInputDisabled(true);
      setTimeout(() => doStartPhase(2, [], acertos, erros), 2000);
    } else if (input === "NAO") {
      const newLines = addLine("Luz da Sala nao esta ligada", lines);
      syncLines(newLines);
      setInputDisabled(true);
      setTimeout(() => doStartPhase(2, [], acertos, erros), 2000);
    } else {
      const newLines = addLine("VOCE E BURRO??", lines);
      syncLines(newLines);
    }
  }

  function handleFase2(input, lines) {
    if (input === "SIM") {
      const newLines = addLine("Luz da Cozinha foi ligada", lines);
      syncLines(newLines);
      setInputDisabled(true);
      setTimeout(() => doStartPhase(3, [], acertos, erros), 2000);
    } else if (input === "NAO") {
      const newLines = addLine("Luz da Cozinha nao foi ligada", lines);
      syncLines(newLines);
      setInputDisabled(true);
      setTimeout(() => doStartPhase(3, [], acertos, erros), 2000);
    } else {
      const newLines = addLine("VOCE E BURRO??", lines);
      syncLines(newLines);
    }
  }

  function handleFase3(input, lines) {
    if (input === "SIM") {
      setInputDisabled(true);
      doStartPhase(4, [], acertos, erros);
    } else if (input === "NAO") {
      setInputDisabled(true);
      doResetAll();
    } else {
      const newLines = addLine("VOCE E BURRO??", lines);
      syncLines(newLines);
    }
  }

  function handleFase4(input, lines) {
    if (input === "CONTINUAR") {
      setInputDisabled(true);
      doStartPhase(5, [], acertos, erros);
    }
  }

  function handleFase5(input, lines) {
    setInputDisabled(true);
    if (input === "14/11" || input === "1") {
      const newAcertos = acertos + 1;
      setAcertos(newAcertos);
      const newLines = addLine("+1 ACERTO", lines);
      syncLines(newLines);
      setTimeout(() => doStartPhase(6, [], newAcertos, erros), 1500);
    } else {
      const newErros = erros + 1;
      setErros(newErros);
      const newLines = addLine("+1 ERRO", lines);
      syncLines(newLines);
      setTimeout(() => doStartPhase(6, [], acertos, newErros), 1500);
    }
  }

  function handleFase6(input, lines) {
    setInputDisabled(true);
    if (input === "SANGUE") {
      const newAcertos = acertos + 1;
      setAcertos(newAcertos);
      const newLines = addLine("+1 ACERTO", lines);
      syncLines(newLines);
      setTimeout(() => doStartPhase(7, [], newAcertos, erros), 1500);
    } else {
      const newErros = erros + 1;
      setErros(newErros);
      const newLines = addLine("+1 ERRO", lines);
      syncLines(newLines);
      setTimeout(() => doStartPhase(7, [], acertos, newErros), 1500);
    }
  }

  function handleFase7(input, lines) {
    setInputDisabled(true);
    if (input === "LAMPIAO" || input === "LAMPIÃO") {
      const newAcertos = acertos + 1;
      setAcertos(newAcertos);
      const newLines = addLine("+1 ACERTO", lines);
      syncLines(newLines);
      setTimeout(() => doPhase8(newAcertos, erros), 1500);
    } else {
      const newErros = erros + 1;
      setErros(newErros);
      const newLines = addLine("+1 ERRO", lines);
      syncLines(newLines);
      setTimeout(() => doPhase8(acertos, newErros), 1500);
    }
  }

  function doPhase8(hits, misses) {
    setInputDisabled(true);
    setPhaseText("");
    setFaseAtual(8);

    const resultLines = [
      `ACERTOS: ${hits}/03   ERROS: ${misses}/03`,
      "",
      "Iniciando desligamento do sistema..."
    ];
    setScreenLines(resultLines);
    screenLinesRef.current = resultLines;

    const duration = hits === 3 ? 60 : hits === 2 ? 110 : 120;
    setLoadingDuration(duration);

    if (stateIdRef.current) {
      base44.entities.TerminalState.update(stateIdRef.current, {
        fase_atual: 8,
        acertos: hits,
        erros: misses,
        screen_lines: resultLines,
        phase_text: "",
        current_input: "",
        is_loading_bar: false,
        loading_duration: duration,
        show_final_message: false,
      }).catch(() => {});
    }

    setTimeout(() => {
      setShowLoadingBar(true);
      if (stateIdRef.current) {
        base44.entities.TerminalState.update(stateIdRef.current, {
          is_loading_bar: true,
          loading_duration: duration,
        }).catch(() => {});
      }
    }, 1500);
  }

  function handleLoadingComplete() {
    setShowLoadingBar(false);
    setShowFinalMessage(true);
    if (stateIdRef.current) {
      base44.entities.TerminalState.update(stateIdRef.current, {
        is_loading_bar: false,
        show_final_message: true,
      }).catch(() => {});
    }
    setTimeout(() => doResetAll(), 5000);
  }

  function doResetAll() {
    setFaseAtual(1);
    setAcertos(0);
    setErros(0);
    setScreenLines([]);
    screenLinesRef.current = [];
    setPhaseText("");
    setInputValue("");
    setShowLoadingBar(false);
    setShowFinalMessage(false);
    setTimeout(() => doStartPhase(1, [], 0, 0), 300);
  }

  function syncLines(lines) {
    if (stateIdRef.current) {
      base44.entities.TerminalState.update(stateIdRef.current, {
        screen_lines: lines,
      }).catch(() => {});
    }
  }

  return (
    <CRTScreen>
      <div
        ref={scrollRef}
        className="h-full overflow-y-auto p-4 md:p-8 pb-20 flex flex-col"
        onClick={() => document.querySelector('input[type="text"]')?.focus()}
      >
        {screenLines.map((line, i) => (
          <TerminalLine key={i} text={line} />
        ))}

        {displayedText && (
          <TerminalLine text={displayedText} isTyping={!isComplete} />
        )}

        {showLoadingBar && (
          <div className="mt-4">
            <LoadingBar
              durationSeconds={loadingDuration}
              onComplete={handleLoadingComplete}
            />
          </div>
        )}

        {showFinalMessage && (
          <div className="mt-6 blink-message glow-text font-terminal text-xl md:text-2xl">
            Alarme de invasao de dados Desligados
          </div>
        )}

        <div className="flex-1 min-h-4" />

        {!showLoadingBar && !showFinalMessage && (
          <div className="mt-4">
            <TerminalInput
              value={inputValue}
              onChange={setInputValue}
              onSubmit={handleSubmit}
              disabled={inputDisabled}
            />
          </div>
        )}
      </div>
    </CRTScreen>
  );
}
