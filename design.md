Role: You are an expert Frontend Developer specializing in high-density documentation UI/UX and the Astro Framework.

Task: Create a responsive site using Tailwind CSS that mirrors the "Google Code Wiki" (codewiki.google) aesthetic based on the Astro Framework. All of the files in /src/content/blog are relevant to this site.

Core Visual Requirements:

    Layout: A three-pane layout.

        Left Sidebar: Collapsible, high-density file tree with 12px font size. Use inter or Roboto font. Include folder icons and chevron toggles.

        Main Content: Central documentation area with a maximum readable width (approx 800px).

        Right Sidebar: "On this page" table of contents and a "Search AI" floating action button.

    Color Palette: >    - Background: Light mode (#ffffff) and Dark mode (#1e1e1e).

        Accents: Google Blue (#4285f4) for active states and links.

        Borders: Subtle 1px dividers using #e0e0e0 (light) or #3c4043 (dark).

    Typography: Use a clean sans-serif for UI (Inter) and a high-legibility monospace for code blocks (JetBrains Mono).

    Components to Include:

        A breadcrumb header (e.g., root > src > components).

        Syntax-highlighted code blocks with a "Copy" button.

        A "Last Updated" metadata footer.

        A search bar at the top that looks like the Google Search Bar (rounded with a subtle shadow).

Technical Constraints: Use Lucide-React for icons. Ensure the layout is a fixed-height viewport where sidebars scroll independently of the main content.