# Money Tracking Application

This is a personal finance tracker built with Svelte that allows users to securely track their income and expenses with a clean dashboard and Google-style UI elements.

## Features
- Add, view, and delete income and expense transactions
- Badge-style category labels for clarity
- Modern confirmation modal (with accessible design) before deleting transactions
- Undo functionality after deletion (optional)
- Visual feedback with styled buttons and tooltips
- Google Firebase authentication and data storage (requires Firebase setup)

## Project Structure
- `src/routes/transactions/+page.svelte`: Main transactions table and delete logic
- `src/routes/dashboard/+page.svelte`: Dashboard with totals and summary
- `src/firebase.js`: Firebase configuration and initialization

## Setup
1. Clone this repository.
2. Run `npm install` to install dependencies.
3. Set up your Firebase project and update `src/firebase.js` with your config.
4. Start the development server with `npm run dev`.

## Usage
- Add a new transaction via the UI.
- Delete a transaction using the trash icon. Confirm deletion in the pop-up modal.
- Each category is shown as a badge for clarity.

## Customization
- Edit badge and modal styles in the `<style>` sections of the Svelte files
- Adjust color schemes or spacing as needed

## License
MIT
