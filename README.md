# 아이디별 저장 기능 추가 가이드

이 저장소에는 아직 사이트 코드가 없어서, 기존 사이트에 붙일 수 있는 **Supabase 기반 로그인·사용자별 저장 구현 가이드**를 정리했습니다.

## 먼저 알아둘 점

GitHub Pages는 정적 호스팅이라 서버 데이터베이스가 없습니다. 같은 브라우저에서만 저장하면 `localStorage`를 쓸 수 있지만, 다른 기기에서도 아이디로 불러오려면 Supabase 같은 백엔드가 필요합니다.

Supabase의 기본 가입은 이메일/비밀번호 인증입니다. 사이트에서 원하는 아이디(사용자명)를 추가로 받도록 할 수 있지만, 이메일 없이 완전히 임의의 아이디와 비밀번호만으로 인증하려면 별도 백엔드 인증을 구현해야 합니다. **사용자명만 입력하는 로그인은 본인 확인이 되지 않아 다른 사람이 같은 이름으로 저장 내용을 볼 수 있으므로 사용하지 마세요.**

## 1. Supabase 프로젝트 준비

1. [Supabase](https://supabase.com/)에서 프로젝트를 만듭니다.
2. 프로젝트의 **Project URL**과 **Publishable key**(또는 구형 프로젝트의 `anon` key)를 확인합니다.
3. 프런트엔드 코드에는 publishable/anon key만 사용합니다. `service_role` 키는 절대로 웹사이트 코드에 넣거나 공개하지 마세요.
4. Authentication 설정에서 이메일/비밀번호 로그인을 활성화합니다.

## 2. 사용자 데이터 테이블과 접근 정책 만들기

Supabase 대시보드의 **SQL Editor**에서 실행합니다. `payload`는 예시 데이터이므로 실제 사이트의 저장 항목에 맞게 컬럼을 바꿀 수 있습니다.

```sql
create table public.user_saves (
  id bigint generated always as identity primary key,
  user_id uuid not null references auth.users(id) on delete cascade,
  save_key text not null,
  payload jsonb not null default '{}'::jsonb,
  updated_at timestamptz not null default now(),
  unique (user_id, save_key)
);

alter table public.user_saves enable row level security;

create policy "Users can read their own saves"
on public.user_saves for select
to authenticated
using ((select auth.uid()) = user_id);

create policy "Users can insert their own saves"
on public.user_saves for insert
to authenticated
with check ((select auth.uid()) = user_id);

create policy "Users can update their own saves"
on public.user_saves for update
to authenticated
using ((select auth.uid()) = user_id)
with check ((select auth.uid()) = user_id);

create policy "Users can delete their own saves"
on public.user_saves for delete
to authenticated
using ((select auth.uid()) = user_id);
```

RLS 정책이 사용자 간 데이터 접근을 막습니다. 프런트엔드에서 사용자 ID를 임의로 지정하는 것만으로는 보안이 되지 않으며, 위 정책을 반드시 적용해야 합니다.

## 3. 사이트에 Supabase 클라이언트 추가

아래 코드를 사이트의 HTML에 연결하거나 기존 JavaScript 모듈에 넣습니다. `YOUR_SUPABASE_URL`과 `YOUR_SUPABASE_PUBLISHABLE_KEY`를 프로젝트 값으로 바꾸세요.

```html
<script type="module">
  import { createClient } from 'https://cdn.jsdelivr.net/npm/@supabase/supabase-js/+esm';

  const supabase = createClient(
    'YOUR_SUPABASE_URL',
    'YOUR_SUPABASE_PUBLISHABLE_KEY'
  );

  // 가입: username은 화면에 표시할 임의 아이디, email/password는 인증 정보입니다.
  async function signUp(username, email, password) {
    const { data, error } = await supabase.auth.signUp({
      email,
      password,
      options: { data: { username } }
    });
    if (error) throw error;
    return data;
  }

  async function signIn(email, password) {
    const { data, error } = await supabase.auth.signInWithPassword({ email, password });
    if (error) throw error;
    return data;
  }

  async function signOut() {
    const { error } = await supabase.auth.signOut();
    if (error) throw error;
  }

  // saveKey별로 현재 로그인한 사용자의 데이터만 저장합니다.
  async function saveForCurrentUser(saveKey, payload) {
    const { data: { user }, error: userError } = await supabase.auth.getUser();
    if (userError) throw userError;
    if (!user) throw new Error('먼저 로그인하세요.');

    const { error } = await supabase
      .from('user_saves')
      .upsert({
        user_id: user.id,
        save_key: saveKey,
        payload,
        updated_at: new Date().toISOString()
      }, { onConflict: 'user_id,save_key' });
    if (error) throw error;
  }

  async function loadForCurrentUser(saveKey) {
    const { data: { user }, error: userError } = await supabase.auth.getUser();
    if (userError) throw userError;
    if (!user) throw new Error('먼저 로그인하세요.');

    const { data, error } = await supabase
      .from('user_saves')
      .select('payload, updated_at')
      .eq('user_id', user.id)
      .eq('save_key', saveKey)
      .maybeSingle();
    if (error) throw error;
    return data?.payload ?? null;
  }

  // 실제 앱의 버튼/저장 데이터에 맞게 연결하세요.
  // 예: await saveForCurrentUser('settings', { theme: 'dark' });
  // 예: const settings = await loadForCurrentUser('settings');

  // 기존 HTML의 다른 모듈에서 호출해야 한다면 window에 노출할 수 있습니다.
  window.userStorage = { signUp, signIn, signOut, saveForCurrentUser, loadForCurrentUser };
</script>
```

## 4. 기존 사이트에 연결하기

- 가입 폼에서 사용자명, 이메일, 비밀번호를 받고 `userStorage.signUp(...)`을 호출합니다.
- 로그인 폼에서 이메일과 비밀번호를 받고 `userStorage.signIn(...)`을 호출합니다.
- 앱의 저장 버튼에서 현재 앱 상태를 객체로 만들어 `saveForCurrentUser('키', 객체)`를 호출합니다.
- 앱이 열릴 때 로그인 상태를 확인한 뒤 `loadForCurrentUser('키')`로 데이터를 불러와 화면에 적용합니다.
- 로그아웃 버튼은 `userStorage.signOut()`에 연결합니다.
- 오류 메시지를 사용자에게 표시하고, 로그인 전에는 저장/불러오기를 막습니다.

## 주의 사항

- 기존 사이트의 저장 데이터 구조와 로그인 UI를 보지 않고는 저장 버튼에 바로 연결되는 완성 파일을 만들 수 없습니다. 이 저장소에는 현재 사이트 코드가 없으므로, 이 문서는 연동 가이드입니다.
- GitHub Pages에서는 프런트엔드 코드와 공개 키를 누구나 볼 수 있습니다. 데이터 보호는 키를 숨기는 방식이 아니라 **인증과 RLS 정책**으로 해야 합니다.
- 가입 후 이메일 확인이 켜져 있으면 확인 전에는 로그인할 수 없을 수 있습니다. 테스트 단계에서는 Supabase Authentication 설정을 확인하세요.
- 이 예시는 저장 내용을 JSON 객체 하나로 저장합니다. 민감한 개인정보나 비밀번호를 `payload`에 저장하지 마세요.
