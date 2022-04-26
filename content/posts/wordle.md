---
author: Taylor Deckard
title: Wordle CLI with Node.js and TypeScript
date: 2022-04-22
description: How to build a Wordle CLI with Node.js and TypeScript
tags: ["programming", "node.js", "typescript"]
---

A few months ago, our team was tasked with leading a new hire programming bootcamp. My focus has been creating course assignments for JavaScript, TypeScript, and Node.js. As a learning exercise, I created a CLI clone of the popular game [Wordle](https://www.nytimes.com/games/wordle/index.html) to use as an example project.

For the bootcamp, I've broken down the project into various learning exercises. These exercises are assessed by unit test cases and linting jobs that run in Concourse CI. The tasks are fairly basic introductory material and (in my opinion) are less interesting than the overall functionality of the project.

For instance, in order to accurately clone the actual Wordle, I did a bit of reverse-engineering that turned out to be fairly simple. Inspecting the Wordle web page JavaScript source reveals that all of the answers for each day are stored in an array of strings. The answer for the current day is determined by using an index derived from the number of days since the first day of Wordle (June 19, 2021.)

With this information, I began coding a program that:
1. Fetches the actual Wordle web page index.html file and parses it to get the JavaScript source file.
```ts
public static parseWordleIndex(html: string) {
  return html.match(/<script src="(main.*?js)"><\/script>/)?.[1] ?? '';
}
```
2. Fetches the JavaScript file from step 1 and parses it to get the list of answers.
```ts
public static parseWordleJavascript(jsFile: string) {
  try {
    const array = jsFile.match(/;var ..=(\[.*?\])/)?.[1];
    return JSON.parse(array ?? '[]');
  } catch (e) {
    return [];
  }
}
```
3. Finds the number of days since June 19, 2021 and uses that to get the current day's answer.
{{< codepen hash="mdpgWBK" user="taylordeckard" >}}

From this point, all that is left is to write the CLI and game logic. I chose to use [commander.js](https://github.com/tj/commander.js), which is a nice wrapper for argument parsing and command execution. Basic setup looks like this:
```typescript
import { program  } from 'commander';
import { commands  } from './commands';

program
  .name('wordle')
  .description('A CLI Wordle clone')
  .version('1.0.0');

// Split commands out into separate modules to keep things tidy.
// Add each command to the program here.
commands.forEach((command) => {
  let pgm = program
    .command(command.name)
    .description(command.description);
  if (command.options?.length) {
    command.options.forEach((opt) => {
      pgm = pgm.option(opt.invocation, opt.description, opt.default);
    });
  }
  pgm = pgm.action(command.action);
});

program.parse(process.argv);
```

The game logic is pretty straight-forward. It give the user 6 chances to guess the correct word. After each guess, log green/yellow colors to indicate correctness.
```typescript
/**
 * Note: This is a snippet from a class method. The DataService
 * is a singleton used to retrieve and store remote data,
 * such as the Wordle answer list.
 */
const ds = DataService.instance;
this._solution = (await ds.solution) ?? '';
// See next code snippet for the Prompter logic
const prompter = new Prompter();
// MAX_ATTEMPTS is 6 
for (let i = 0; i < MAX_ATTEMPTS; i += 1) {
  // Pass in the number of attempts remaining to inform the user
  const { guess } = await prompter.promptUserGuess(MAX_ATTEMPTS - i);
  // Print results
  new Guess(guess, this._solution)
	.markGreen()
	.markYellow()
	.logOutput();

  if (guess === this._solution) {
	Logger.printf(YOU_WIN);
	this._solved = true;
	break;
  }
}
if (!this._solved) {
  Logger.printf(YOU_LOSE, this._solution);
}
```

For prompting the user, I chose to use the [inquirer.js](https://github.com/SBoudrias/Inquirer.js) module. It allows for input transformation and validation. 
```typescript
public async promptUserGuess(attemptsRemaining: number) {
  if (!this._acceptableGuesses) {
    const ds = DataService.instance;
    this._acceptableGuesses = await ds.wordlist;
  }

  return inquirer.prompt([
    {
      message: `Guess a 5-letter word (${attemptsRemaining} attempts remaining):`,
      name: 'guess',
      transformer: (input: string) => input.toLowerCase(),
      type: 'input',
      validate: Validator.checkGuess.bind(this, this._acceptableGuesses),
    },
  ]);
}
```

Finally, the logic for colorizing the guesses is:
```typescript
/**
 * Marks letters of the guess as green (correct)
 *
 * @returns {Guess} guess
 */
public markGreen() {
  // iterate over each letter of the guess
  for (let i = 0; i < 5; i += 1) {
    const guessChar = this._guess[i];
    const actualChar = this._solution[i];
    // if the character in the guess matches the character at the
    // same index of the solution, it should be marked green
    if (guessChar === actualChar) {
      this._output[i] = Colorizer.green(guessChar);
      this._correctness[i] = Correctness.GREEN;
      this._remainingChars[i] = '';
    }
  }
  return this;
}

/**
 * Marks letters of the guess as yellow (included in solution but wrong index)
 *
 * @returns {Guess} guess
 */
public markYellow() {
  // iterate over each letter of the guess
  for (let i = 0; i < 5; i += 1) {
    const guessChar = this._guess[i];
    /* if the character is included in the _remainingChars and does not
       match the current index of the solution, mark yellow
       note:  _remainingChars is an array of characters of the solution
       that have not been marked green */
    if (this._remainingChars.includes(guessChar) && guessChar !== this._solution[i]) {
      this._correctness[i] = Correctness.YELLOW;
      this._output[i] = Colorizer.yellow(guessChar);
    }
  }
  return this;
}
```

Now it's done. The program can be executed with:
```sh
npm start -- start
```

Later, I started working on another command that tries to solve the Wordle puzzle using an algorithm. For a start, I filtered the list of possible words based on previous guess outcomes. Then, I tested the algorithm against the list of wordle answers. I began tweaking the algorithm from there, doing things like trying different starting words or using strategies to eliminate letters.

![A few hours later](/blog/images/wordle/a_few_hours_later.jpeg)

I was able adjust enough to achieve ~99.61% win accuracy. The automation can be run with:
```sh
npm start -- solve
```

I'm sure the algorithm can be improved to reach 100%, though it's a task for another time.

[Check out the project on Github.](https://github.com/taylordeckard/node-wordle-cli)
