UI Preferences and Structure Guidelines

This document specifies the mandatory aesthetic, responsiveness, and component organization standards for the Next.js frontend. All development must strictly adhere to these guidelines.

1. General UI/UX Recommendations

The application must adopt a modern, minimal CRM-style layout with a focus on usability and clarity.

Layout: Use a clean layout featuring a collapsible sidebar for navigation. The main content area must be centered, padded, and utilize rounded cards with subtle shadows for visual separation (shadow-md, shadow-lg).

Typography: Typography should be bold for headings, with clear hierarchy, ample spacing, and readable colors (text-gray-900, text-gray-700). Use font-extrabold, font-bold, font-medium, and font-semibold for emphasis.

Aesthetics: Prefer gradients and soft color backgrounds for dashboard landing pages (bg-gradient-to-br, bg-indigo-100).

Navigation: Sidebar navigation must use icons and labels (lucide-react), feature active state highlighting, and use smooth transitions (transition-all).

Buttons: Use color-coded buttons (e.g., blue for primary actions, purple for uploads) with rounded-lg corners and hover transitions (hover:bg-blue-700, hover:bg-purple-700). Use disabled:opacity-50 and disabled:cursor-not-allowed for disabled states.

Modals: Modals must be centered, utilize a backdrop blur (backdrop-blur-sm) and shadow, feature rounded corners, and include clear close buttons. Use z-50 for modal overlays and max-h-[90vh] with overflow-y-auto for content.

Forms: Forms should use full-width inputs, rounded borders (rounded-md), and clear focus/hover states (focus:ring, focus:outline-none).

Icons: Use icons from lucide-react for all visual cues (e.g., Box, ShoppingBag, PanelLeftClose/Open).

Accessibility: All UI elements must be accessible, with sufficient color contrast and keyboard navigation support.

Spacing: Use space-y and gap classes extensively for vertical and horizontal spacing.

Depth and Separation: Utilize shadow-lg, shadow-xl, and shadow-inner for depth. Use divide-y and border classes for clear separation in lists and tables.

Transitions: Use transition-all and transition-colors for smooth UI state changes.

Animation: Use animation classes (e.g., animate-scale-in) for modal transitions and interactive elements.

2. Tables and Data Display

Responsiveness: Tables must be fully responsive and scale appropriately across devices.

Styling: Include sticky headers, zebra striping (bg-gray-50 rows), and clear, uppercase column labels with wide tracking.

Data Actions: Pagination controls should be rounded, with clear disabled states and navigation.

Uploads: CSV upload buttons must be prominent, rounded buttons with clear labels.

3. Responsive Design

The layout must be fully adaptable for mobile, tablet, and desktop using Tailwind's responsive classes (sm:, md:, lg:).

The sidebar must collapse to icons-only on small screens or when toggled to maximize content space.

Use max-w-xs, max-w-2xl, min-h-full, and min-w-full for responsive sizing.

4. Theme Management

Light/Dark Mode: Support both light and dark mode using Tailwind's dark: classes.

Persistence: The theme preference must be persisted in localStorage.

5. Component Structure

The project must strictly adhere to the following directory structure for the Next.js application:

shared/: Reusable, generic components used across the entire application (e.g., navigation bar, modals, buttons).

dashboard/: Components specific to the CRM dashboard (cards, tables, entity-specific forms/modals, sidebar).

lib/: Utility functions and external library configs.

middleware.ts: Central file for authorization logic and role-based access checks.

prisma/: Prisma schema and related database migration files.

public/: Static assets (images, fonts).

types/: Centralized TypeScript types and interfaces.
