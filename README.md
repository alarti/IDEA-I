--- Author: Alberto Arce

# IDEA-I Cloud: AI-Assisted IDE

IDEA-I is an experimental, AI-powered IDE that allows you to build and preview web applications through a conversational interface. It features a file explorer, a real-time code editor, and a live preview panel, all enhanced with generative AI capabilities to assist in development.

This version is integrated with Supabase for user authentication, data persistence, and real-time synchronization, allowing for private user projects, public/guest projects, and an administrator role.

## Supabase Configuration

To use the authentication and cloud synchronization features, you must configure a Supabase project.

### 1. Create a Supabase Project

1.  Go to [supabase.com](https://supabase.com/) and create a new account or sign in.
2.  Create a new project. Save your **Project URL** and **anon public key**, as you will need them later.

### 2. Database Setup (Full Reset Script)

The following script will completely reset your project's database. It deletes all existing tables, functions, and policies before recreating them. This is useful for ensuring your schema is perfectly in sync with the application.

**⚠️ WARNING: This is a destructive operation and will permanently delete all data in your `projects`, `files`, `tasks`, `test_runs`, and `console_logs` tables.**

Go to the **SQL Editor** in your Supabase project dashboard, paste the entire script below, and click "Run".

```sql
-- =============================================
-- ===          PART 1: TEARDOWN             ===
-- =============================================
-- This section drops all existing objects to ensure a clean slate.

-- Disable Row Level Security to drop policies
ALTER TABLE projects DISABLE ROW LEVEL SECURITY;
ALTER TABLE files DISABLE ROW LEVEL SECURITY;
ALTER TABLE tasks DISABLE ROW LEVEL SECURITY;
ALTER TABLE test_runs DISABLE ROW LEVEL SECURITY;
ALTER TABLE console_logs DISABLE ROW LEVEL SECURITY;

-- Drop existing policies if they exist
DROP POLICY IF EXISTS "Allow admin full access on console_logs" ON console_logs;
DROP POLICY IF EXISTS "Allow read access to console_logs in accessible projects" ON console_logs;
DROP POLICY IF EXISTS "Allow write access to console_logs in own projects" ON console_logs;
DROP POLICY IF EXISTS "Allow admin full access on test_runs" ON test_runs;
DROP POLICY IF EXISTS "Allow read access to test_runs in accessible projects" ON test_runs;
DROP POLICY IF EXISTS "Allow write access to test_runs in own projects" ON test_runs;
DROP POLICY IF EXISTS "Allow admin full access on tasks" ON tasks;
DROP POLICY IF EXISTS "Allow read access to tasks in accessible projects" ON tasks;
DROP POLICY IF EXISTS "Allow write access to tasks in own projects" ON tasks;
DROP POLICY IF EXISTS "Allow admin full access on files" ON files;
DROP POLICY IF EXISTS "Allow read access to files in accessible projects" ON files;
DROP POLICY IF EXISTS "Allow write access to files in own projects" ON files;
DROP POLICY IF EXISTS "Allow admin full access on projects" ON projects;
DROP POLICY IF EXISTS "Allow read access to own and public projects" ON projects;
DROP POLICY IF EXISTS "Allow authenticated users to create projects" ON projects;
DROP POLICY IF EXISTS "Allow users to update their own projects" ON projects;
DROP POLICY IF EXISTS "Allow users to delete their own projects" ON projects;

-- Drop triggers
DROP TRIGGER IF EXISTS update_projects_last_modified ON projects;
DROP TRIGGER IF EXISTS update_files_last_modified ON files;

-- Drop helper functions for policies before dropping tables
DROP FUNCTION IF EXISTS is_project_owner(uuid);
DROP FUNCTION IF EXISTS can_access_project(uuid);

-- Drop tables in reverse order of dependency
DROP TABLE IF EXISTS console_logs;
DROP TABLE IF EXISTS test_runs;
DROP TABLE IF EXISTS tasks;
DROP TABLE IF EXISTS files;
DROP TABLE IF EXISTS projects;

-- Drop other functions
DROP FUNCTION IF EXISTS update_last_modified_column();
DROP FUNCTION IF EXISTS is_admin();


-- =============================================
-- ===           PART 2: SETUP               ===
-- =============================================
-- This section recreates the entire database schema from scratch.

-- Create the projects table
CREATE TABLE projects (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid REFERENCES auth.users(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  initialPrompt TEXT,
  autonomyLevel SMALLINT DEFAULT 3 NOT NULL,
  createdAt TIMESTAMPTZ DEFAULT now() NOT NULL,
  author TEXT,
  aiProvider TEXT DEFAULT 'gemini' NOT NULL,
  ollamaApiUrl TEXT,
  ollamaModelName TEXT,
  is_public BOOLEAN DEFAULT false NOT NULL,
  last_modified TIMESTAMPTZ DEFAULT now() NOT NULL
);

-- Create the files table
CREATE TABLE files (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  projectId uuid REFERENCES projects(id) ON DELETE CASCADE NOT NULL,
  user_id uuid REFERENCES auth.users(id) ON DELETE CASCADE,
  path TEXT NOT NULL,
  content TEXT,
  last_modified TIMESTAMPTZ DEFAULT now() NOT NULL,
  -- Ensure no duplicate file paths within a project
  UNIQUE(projectId, path)
);

-- Create the tasks table
CREATE TABLE tasks (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    projectId uuid REFERENCES projects(id) ON DELETE CASCADE NOT NULL,
    user_id uuid REFERENCES auth.users(id) ON DELETE CASCADE,
    description TEXT NOT NULL,
    isComplete BOOLEAN DEFAULT false NOT NULL,
    createdAt TIMESTAMPTZ DEFAULT now() NOT NULL
);

-- Create the test_runs table
CREATE TABLE test_runs (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    projectId uuid REFERENCES projects(id) ON DELETE CASCADE NOT NULL,
    user_id uuid REFERENCES auth.users(id) ON DELETE CASCADE,
    results JSONB,
    passed_count INT NOT NULL,
    failed_count INT NOT NULL,
    total_duration FLOAT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now() NOT NULL
);

-- Create the console_logs table
CREATE TABLE console_logs (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    projectId uuid REFERENCES projects(id) ON DELETE CASCADE NOT NULL,
    user_id uuid REFERENCES auth.users(id) ON DELETE CASCADE,
    level TEXT NOT NULL,
    message JSONB,
    created_at TIMESTAMPTZ DEFAULT now() NOT NULL
);

-- Create a function to automatically update the `last_modified` timestamp
CREATE OR REPLACE FUNCTION update_last_modified_column()
RETURNS TRIGGER AS $$
BEGIN
   NEW.last_modified = now();
   RETURN NEW;
END;
$$ language 'plpgsql';

-- Create triggers to call the function when a row is updated
CREATE TRIGGER update_projects_last_modified
BEFORE UPDATE ON projects
FOR EACH ROW
EXECUTE FUNCTION update_last_modified_column();

CREATE TRIGGER update_files_last_modified
BEFORE UPDATE ON files
FOR EACH ROW
EXECUTE FUNCTION update_last_modified_column();

-- Add indexes for better query performance
CREATE INDEX idx_files_project_id ON files(projectId);
CREATE INDEX idx_tasks_project_id ON tasks(projectId);
CREATE INDEX idx_test_runs_project_id ON test_runs(projectId);
CREATE INDEX idx_console_logs_project_id ON console_logs(projectId);
CREATE INDEX idx_projects_user_id ON projects(user_id);
CREATE INDEX idx_projects_is_public ON projects(is_public);

-- Helper function to check for admin role
-- ❗️ Replace with the actual UUID of your admin user from the `auth.users` table.
CREATE OR REPLACE FUNCTION is_admin()
RETURNS boolean
LANGUAGE sql
SECURITY DEFINER
AS $$
  SELECT auth.uid() = '1dd4fd82-e3f9-40ef-b628-cb3decd55881'::uuid;
$$;


-- =============================================
-- ===   PART 3: ROW LEVEL SECURITY (RLS)    ===
-- =============================================
-- This section enables and configures all security policies.

-- Enable Row Level Security on all tables
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;
ALTER TABLE files ENABLE ROW LEVEL SECURITY;
ALTER TABLE tasks ENABLE ROW LEVEL SECURITY;
ALTER TABLE test_runs ENABLE ROW LEVEL SECURITY;
ALTER TABLE console_logs ENABLE ROW LEVEL SECURITY;

-- Policies for 'projects' table
CREATE POLICY "Allow admin full access on projects" ON projects FOR ALL USING (is_admin()) WITH CHECK (is_admin());
CREATE POLICY "Allow read access to own and public projects" ON projects FOR SELECT USING (auth.uid() = user_id OR is_public = true);
CREATE POLICY "Allow authenticated users to create projects" ON projects FOR INSERT WITH CHECK (auth.uid() = user_id);
CREATE POLICY "Allow users to update their own projects" ON projects FOR UPDATE USING (auth.uid() = user_id) WITH CHECK (auth.uid() = user_id);
CREATE POLICY "Allow users to delete their own projects" ON projects FOR DELETE USING (auth.uid() = user_id);

-- Helper function to check project ownership/visibility (used by child tables)
CREATE OR REPLACE FUNCTION can_access_project(project_id uuid)
RETURNS boolean
LANGUAGE sql
SECURITY DEFINER
AS $$
  SELECT EXISTS (
    SELECT 1 FROM projects
    WHERE id = project_id AND (user_id = auth.uid() OR is_public = true OR is_admin())
  );
$$;

-- Helper function to check project ownership for writing
CREATE OR REPLACE FUNCTION is_project_owner(project_id uuid)
RETURNS boolean
LANGUAGE sql
SECURITY DEFINER
AS $$
  SELECT EXISTS (
    SELECT 1 FROM projects
    WHERE id = project_id AND (user_id = auth.uid() OR is_admin())
  );
$$;

-- Policies for 'files' table
CREATE POLICY "Allow admin full access on files" ON files FOR ALL USING (is_admin()) WITH CHECK (is_admin());
CREATE POLICY "Allow read access to files in accessible projects" ON files FOR SELECT USING (can_access_project(projectId));
CREATE POLICY "Allow write access to files in own projects" ON files FOR ALL USING (is_project_owner(projectId)) WITH CHECK (is_project_owner(projectId));

-- Policies for 'tasks' table
CREATE POLICY "Allow admin full access on tasks" ON tasks FOR ALL USING (is_admin()) WITH CHECK (is_admin());
CREATE POLICY "Allow read access to tasks in accessible projects" ON tasks FOR SELECT USING (can_access_project(projectId));
CREATE POLICY "Allow write access to tasks in own projects" ON tasks FOR ALL USING (is_project_owner(projectId)) WITH CHECK (is_project_owner(projectId));

-- Policies for 'test_runs' table
CREATE POLICY "Allow admin full access on test_runs" ON test_runs FOR ALL USING (is_admin()) WITH CHECK (is_admin());
CREATE POLICY "Allow read access to test_runs in accessible projects" ON test_runs FOR SELECT USING (can_access_project(projectId));
CREATE POLICY "Allow write access to test_runs in own projects" ON test_runs FOR ALL USING (is_project_owner(projectId)) WITH CHECK (is_project_owner(projectId));

-- Policies for 'console_logs' table
CREATE POLICY "Allow admin full access on console_logs" ON console_logs FOR ALL USING (is_admin()) WITH CHECK (is_admin());
CREATE POLICY "Allow read access to console_logs in accessible projects" ON console_logs FOR SELECT USING (can_access_project(projectId));
CREATE POLICY "Allow write access to console_logs in own projects" ON console_logs FOR ALL USING (is_project_owner(projectId)) WITH CHECK (is_project_owner(projectId));

-- Grant usage on new functions to the authenticated role
GRANT EXECUTE ON FUNCTION is_admin() TO authenticated;
GRANT EXECUTE ON FUNCTION can_access_project(uuid) TO authenticated;
GRANT EXECUTE ON FUNCTION is_project_owner(uuid) TO authenticated;

```

### 3. Authentication Setup

#### Step 3.1: Enable Google Provider

1.  In your Supabase dashboard, go to **Authentication** -> **Providers**.
2.  Enable the **Google** provider. You will likely need to create OAuth credentials in the Google Cloud Console and add the Client ID and Client Secret here.
3.  Ensure you add the Supabase redirect URL provided in the Google provider settings to your Google Cloud OAuth configuration.

#### Step 3.2: Configure Application Credentials

1.  Open the file `services/supabase.ts` in this project.
2.  Replace the placeholder values for `supabaseUrl` and `supabaseAnonKey` with the credentials you saved from your Supabase project's API settings.
3.  Sign up to the application using the Google account you wish to be the administrator.
4.  Go to the Supabase Table Editor, open the `auth.users` table, and copy the `id` (UUID) of your admin user.
5.  Paste this UUID into the `ADMIN_USER_ID` constant in `services/supabase.ts`.
6.  **Crucially**, also paste this same UUID into the `is_admin()` SQL function you created in Step 2 and run it again to update it.

After completing these steps, your application should be fully configured to work with Supabase.