<script lang="ts">
    import { onMount } from 'svelte';
    import { blur } from 'svelte/transition'
    import { tweened } from 'svelte/motion';

    type Game = 'waiting for input' | 'in progress' | 'finished';
    type Word = string;

    let game: Game = 'waiting for input';
    let seconds = 1000;
    let typedLetter = '';

    let words: Word[] = [];

    let wordIndex = 0;
    let letterIndex = 0;
    let correctLetters = 0;
    let toggleReset = false;

    let wordPerMinute = tweened(0, { delay: 300, duration: 1000 });
    let accuracy = tweened(0, { delay: 1300, duration: 1000 });



    let wordsEl: HTMLDivElement;
    let letterEl: HTMLSpanElement;
    let inputEl: HTMLInputElement;
    let caretEl: HTMLDivElement;

    function resetGame() {
        toggleReset = !toggleReset;

        setGameState('waiting for input');
        getWords(100);

        seconds = 30;
        typedLetter = '';
        wordIndex =  0;
        letterIndex = 0;
        correctLetters = 0;

        $wordPerMinute = 0;
        $accuracy = 0;
    }

    function getWordsPerMinute() {
        const word = 5;
        const minutes = 0.5;
        return Math.floor(correctLetters / word / minutes);
    }

    function getAccuracy() {
        const totalLetters = getTotalLetters(words);
        return Math.floor(correctLetters / totalLetters * 100);
    }

    function getTotalLetters(words: Word[]) {
        return words.reduce((acc, word) => acc + word.length, 0);
    }

    function getResults() {
        $wordPerMinute = getWordsPerMinute();
        $accuracy = getAccuracy();
    }
    function updateGameState() {

        if (!typedLetter.match(/[a-zA-Z]/g)) {
            return;
        }

        setLetter();
        checkLetter();
        nextLetter();
        updateLine();
        resetLetter();
        moveCaret();
    }

    function setLetter() {
        const isWordComplete = letterIndex > words[wordIndex].length - 1;
        
        if(!isWordComplete) {
            letterEl = wordsEl.children[wordIndex].children[letterIndex] as HTMLSpanElement;
        }
    }

    function previousSetLetter() {
        const isWordComplete = letterIndex > words[wordIndex].length - 1;

        if(!isWordComplete) {
            letterEl = wordsEl.children[wordIndex].children[letterIndex - 1] as HTMLSpanElement;
        }
    }

    function checkLetter() {
        const currentLetter = words[wordIndex][letterIndex];

        if(typedLetter === currentLetter) {
            letterEl.dataset.letter = 'correct';
            increaseScore();
        }

        if(typedLetter !== currentLetter) {
            letterEl.dataset.letter = 'incorrect';
        }
    }

    function deleteCheckLetter() {

        letterEl.dataset.letter = '';
    }

    function increaseScore() {
        correctLetters += 1;
    }

    function nextLetter() {
        letterIndex += 1;
    }

    function previousLetter() {
        letterIndex -= 1;        
    }


    //TODO: I understand this.
    function updateLine() {
        const wordEl = wordsEl.children[wordIndex];
        const wordsY = wordsEl.getBoundingClientRect().y;
        const wordY = wordEl.getBoundingClientRect().y;
        
        if(wordY > wordsY) {
            wordEl.scrollIntoView({block: 'center'});
        }
    }

    function resetLetter() {
        typedLetter = '';
    }

    function moveCaret() {
        const offset = 4;
        const { offsetLeft, offsetTop, offsetWidth } = letterEl;

        caretEl.style.top = `${offsetTop + offset}px`;
        caretEl.style.left = `${offsetLeft + offsetWidth}px`;
    }

    function  previousCaret() {
        const offset = 4;

        if(!letterEl) return;

        const { offsetLeft , offsetTop, offsetWidth } = letterEl;
        
        caretEl.style.top = `${offsetTop + offset}px`;
        caretEl.style.left = `${ offsetLeft - offsetWidth }px`;
    }

    function nextWord() {
        const isNotFirstLetter  = letterIndex !== 0;
        const isOneLetterWord   = words[wordIndex].length === 1;

        if(isNotFirstLetter || isOneLetterWord) {
            wordIndex += 1;
            letterIndex = 0;
            increaseScore();
            moveCaret();
        }
    }
    

    function startGame() {
        setGameState('in progress');
        setGameTimer();
    }

    function setGameState(state: Game) {
        game = state;
    }

    function setGameTimer() {
        const gameTimer = () => {
            if(seconds > 0) {
                seconds -= 1;
            }
            
            if(game === 'waiting for input' || seconds  === 0) {
                clearInterval(interval);
            }

            if(seconds === 0) {
                setGameState('finished');
                getResults();
            }
        }
        const interval = setInterval(gameTimer, 1000);
    }


    async function getWords(limit: number) {
        const response = await fetch(`/api/words?limit=${limit}`);
        words = await response.json();
    }

    function focusInput() {
        if( inputEl ) {
            inputEl.focus();
        }
    }

    function handleKeyDown(event: KeyboardEvent) {
        if (event.code === 'Space'){
            event.preventDefault();
            
            if(game === 'in progress') {
                nextWord();
            }
        }


        if (event.code === 'Backspace' && letterIndex >= 1) {
            event.preventDefault();

            if (game === 'in progress') {
                previousSetLetter();
                deleteCheckLetter();
                previousLetter();
                // previousCaret();
            }
        }

        if (game === 'waiting for input') {
            startGame();
        }
    }

    onMount(() => {
        getWords(100);
        focusInput();
    });
</script>


<svelte:body
    on:click={focusInput}
/>


{#if game !== 'finished'}
    <div class="game" data-game={game}>
        <input 
            bind:this={inputEl}
            bind:value={typedLetter}
            on:input={updateGameState}
            on:keydown={handleKeyDown}
            class="input"
            type="text"
        />

    <div class="time">{seconds}</div>

    {#key toggleReset}
        <div in:blur|local bind:this={wordsEl} class="words">
            {#each words as word}
            <span class="word">
                {#each word as letter}
                <span class="letter">{letter}</span>
                {/each}
            </span>
            {/each}

            <div bind:this={caretEl} class="caret">
            </div>
        </div>
    {/key}

    <div class="reset">
        <button on:click={resetGame} aria-label="reset">
            <svg
              xmlns="http://www.w3.org/2000/svg"
              viewBox="0 0 24 24"
              width="24"
              height="24"
              stroke-width="1.5"
              stroke="currentColor"
              fill="none"
            >
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                d="M15 15l6-6m0 0l-6-6m6 6H9a6 6 0 000 12h3"
              />
            </svg>
        </button>
    </div>
</div>
{/if}

{#if game === 'finished'}
    <div in:blur class="results">
        <div>
            <p class="title">wpm</p>
            <p class="score">{Math.trunc($wordPerMinute)}</p>
        </div>
        <div>
            <p class="title">accuracy</p>
            <p class="score">{Math.trunc($accuracy)}%</p>
        </div>
    </div>
{/if}

<style lang="scss">

    .game {
        position: relative;
        
        .input {
            position: absolute;
            opacity: 0;
        }
        
        .time {
            position: absolute;
            top: -48px;
            font-size: 1.5rem;
            color: var(--primary-color);
            opacity: 0;
            transition: opacity 0.3s ease-in-out;
        }
        

        &[data-game="in progress"] .time {
            opacity: 1;
        }


        .reset {
            width: 100%;
            display: grid;
            justify-content: center;
            margin-top: 2rem;
        }
    }
    .words {
        --line-height: 1rem;
        --lines: 3;

        width: 100%;
        max-height: calc(var(--line-height) * var(--lines) * 1.42);
        display: flex;
        flex-wrap: wrap;
        gap: 0.6em;
        position: relative;
        font-size: 1.5rem;
        line-height: var(--line-height);
        overflow: hidden;
        user-select: none;

        .letter {
            opacity: 0.4;
            transition: opacity 0.3s ease;

            &:global([data-letter="correct"]) {
                opacity: 0.8;
            }

            &:global([data-letter="incorrect"]) {
                color: var(--primary-color);
                opacity: 1;
            }
        }

        .caret {
            position: absolute;
            height: 1.8rem;
            top: 0;
            border-right: 1px solid var(--primary-color);
            animation: caret 1s infinite;
            transition: all 0.2s ease;

            @keyframes caret {
                0%, 
                to {
                    opacity: 0;
                }
                
                50% {
                    opacity: 1;
                }
            }
        }
    }

    .results {
        .title {
            font-size: 2rem;
            color: var(--fg-200);
        }

        .score {
            font-size: 4rem;
            color: var(--primary-color);
        }

        .play {
            margin-top: 1rem;
        }
    }
</style>