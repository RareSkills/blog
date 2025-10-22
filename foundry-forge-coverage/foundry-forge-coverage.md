# Foundry forge coverage

## Visual line coverage report with LCOV

![forge coverage lcov report](https://static.wixstatic.com/media/935a00_95a73f5f70b24200a5bde10b3a0243b6~mv2.webp/v1/fill/w_740,h_178,al_c,q_80,usm_0.66_1.00_0.01,enc_auto/935a00_95a73f5f70b24200a5bde10b3a0243b6~mv2.webp)

If you run "forge coverage" in a [Foundry](https://book.getfoundry.sh/) project, you'll get a table showing how much of your lines and branches are covered.

![Foundry forge coverage](https://static.wixstatic.com/media/935a00_ed8796a9a0814f54a73a0844cd7675f0~mv2.webp/v1/fill/w_740,h_216,al_c,q_80,usm_0.66_1.00_0.01,enc_auto/935a00_ed8796a9a0814f54a73a0844cd7675f0~mv2.webp)

If you want to see visually which lines and branches are or are not covered, use the following steps

## Instructions to get line visual coverage in Foundry

### 1. Install genhtml

```shell
brew install genhtml
```

### 2. Create a coverage directory in your Foundry project

```shell
mkdir coverage
```

### 3. Run the following command

```shell
forge coverage --report lcov && genhtml lcov.info --branch-coverage --output-dir coverage
```

### 4. Open the following file

```shell
coverage/index.html
```

And you'll be able to see a coverage report like what you see at the top of the page.

### No available formula with the name "genhtml". Did you mean ekhtml?

If you get this error, do:

```shell
brew install ekhtml
```

and it will install genhtml.

### Credits

This tutorial was created by Matteo Vendittoli, a student at the RareSkills [blockchain bootcamp](https://www.rareskills.io/web3-blockchain-bootcamps).

## Addendum: Visual line coverage in Visual Studio Code

It is also possible to view line coverage directly inside Visual Studio Code. Credit to [0xasp_](https://twitter.com/0xasp_) on Twitter for sharing this with us ([original tweet](https://twitter.com/0xasp_/status/1624456443327029248)).

### 1. Install the Coverage Gutters extension in VSCode

### 2. Generate the coverage report

```shell
forge coverage --report lcov
```

### 3. Open the Command Palette

Pick display coverage report

*Originally Published Feb 8, 2023*
