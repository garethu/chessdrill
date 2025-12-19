# Chess Drill Trainer

A web-based chess opening trainer that helps you practice and memorize chess opening lines using PGN files. Built with Vue 3, TypeScript, and Chessground (the same chess board library used by Lichess.org).

![Chess Drill Trainer](https://img.shields.io/badge/chess-trainer-green)
![Vue 3](https://img.shields.io/badge/vue-3.5-blue)
![TypeScript](https://img.shields.io/badge/typescript-5.9-blue)

## Features

✨ **Interactive Chess Board** - Drag and drop pieces with legal move highlighting

🎯 **PGN Support** - Load multiple opening lines from standard PGN files

🤖 **Automated Opponent Moves** - The computer automatically plays opponent moves

✅ **Move Validation** - Instant feedback with red/green arrows for incorrect/correct moves

💡 **Hint System** - Get hints showing the correct move when you're stuck

🎨 **Beautiful UI** - Clean, modern interface with smooth animations

🎉 **Completion Dialog** - Celebrates when you successfully complete a variation

⚫⚪ **Play Either Color** - Choose to play as White or Black

## How to Use

### 1. Load Your Opening Lines

Click the **"Load PGN"** button and select a PGN file containing your opening lines. The app supports:
- Multiple games in one file
- Each game becomes a separate training line
- Standard PGN format with Event tags

**Example PGN format:**
```pgn
[Event "King's Gambit - Main Line"]
[White "?"]
[Black "?"]
[Result "*"]

1. e4 e5 2. f4 exf4 3. Nf3 g5 4. h4
```

### 2. Select Your Line

If your PGN contains multiple lines, use the **"Select Line"** dropdown to choose which opening you want to practice.

### 3. Choose Your Color

Use the **"Play as"** dropdown to select whether you want to play as **White** or **Black**.

### 4. Start Training

Click the **"Train"** button to begin:
- If playing **Black**: White automatically makes the first move
- If playing **White**: Make your first move
- Legal moves are shown with dots when you pick up a piece

### 5. Practice the Line

- **Correct Move**: ✅ Opponent responds automatically after 0.5 seconds
- **Wrong Move**: ❌ Move is undone, red arrow shows your move, green arrow shows correct move (displays for 2 seconds)
- **Stuck?**: Click the **"Hint"** button to see the correct move with a blue arrow

### 6. Complete the Variation

When you successfully complete all moves in the variation, you'll see a congratulations dialog! Click **"Continue Training"** to reset and practice again.

## Development Setup

### Prerequisites

- Node.js (v16 or higher)
- npm or yarn

### Installation

1. Clone the repository:
```bash
git clone <your-repo-url>
cd chessdrill-gemini
```

2. Install dependencies:
```bash
npm install
```

3. Start the development server:
```bash
npm run dev
```

4. Open your browser to `http://localhost:5173`

### Build for Production

```bash
npm run build
```

The production files will be in the `dist` folder.

## Deployment

### Web Server Deployment

1. Build the production version:
```bash
npm run build
```

2. Upload the contents of the `dist` folder to your web server:
   - `index.html`
   - `assets/` folder
   - `img/` folder
   - `vite.svg`

3. Access via your domain - the app is entirely client-side and requires no server configuration.

### Hosting Options

Works on any static hosting service:
- GitHub Pages
- Netlify
- Vercel
- Traditional web hosting via FTP
- AWS S3 + CloudFront
- Firebase Hosting

## Sample PGN File

A sample PGN file (`rousseau-gambit.pgn`) is included in the `src/pgn/` folder with 12 variations of the Rousseau Gambit opening.

## Technologies Used

- **Vue 3** - Progressive JavaScript framework
- **TypeScript** - Type-safe JavaScript
- **Chessground** - Professional chess board component (used by Lichess.org)
- **chess.js** - Chess move validation and game logic
- **@mliebelt/pgn-parser** - PGN file parsing
- **Vite** - Fast build tool and dev server

## Project Structure

```
chessdrill-gemini/
├── src/
│   ├── components/
│   │   └── ChessTrainer.vue    # Main chess trainer component
│   ├── pgn/
│   │   └── rousseau-gambit.pgn # Sample PGN file
│   ├── App.vue                 # Root component
│   └── main.ts                 # Application entry point
├── public/                     # Static assets
├── dist/                       # Production build output
└── package.json                # Dependencies
```

## Tips for Best Experience

1. **Create Quality PGN Files**: Use clear Event tags to name your lines
2. **Practice Regularly**: Repetition is key to memorizing openings
3. **Use Hints Sparingly**: Try to recall moves before using hints
4. **Focus on One Line**: Master one variation before moving to the next
5. **Play Both Colors**: Practice from both White and Black perspectives

## Troubleshooting

### Pieces Won't Move
- Make sure you clicked the "Train" button
- Check that it's your turn (correct color for current position)

### PGN Won't Load
- Ensure your PGN file is valid standard format
- Check the browser console (F12) for parsing errors
- Each game should have at least an Event tag and moves

### Wrong Moves Not Detected
- The app validates against the exact move sequence in your PGN
- Alternative variations are not currently supported within a single game

## Future Enhancements

Potential features for future versions:
- Spaced repetition system for optimal learning
- Statistics tracking (success rate, time per move)
- Multiple variation support within single games
- Move annotations and explanations
- Sound effects for moves
- Mobile-responsive design improvements

## License

MIT License - Feel free to use and modify for your own chess training!

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## Acknowledgments

- Lichess.org for the excellent Chessground library
- The chess.js library for robust chess logic
- Vue.js community for the amazing framework

---

**Happy Training! May your openings be sharp and your tactics sound!** ♟️
