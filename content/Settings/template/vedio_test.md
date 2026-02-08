<%* 
const LESSON_SEASON_KEY = 'ugc_season';

function transform_page_to_chapter_str(page, view_url) {
  return "[" + page.part + "](" + view_url + "?p=" + String(page.page) + ")";
}

function deleteIllegalChars(s) {
  // 把常见 Windows / macOS 非法字符都换成空格，再去掉多余空格
  return s.replace(/[/\\|*?"<>:\s]/g, ' ').replace(/\s+/g, ' ').trim();
}

function transformTitleToChapterStr(title, bvid) {
  return `[${title}](https://www.bilibili.com/video/${bvid})`;
}

function get_chapters_from_ugc_season(respData) {
  const episodes = respData[LESSON_SEASON_KEY].sections[0].episodes;
  const chapterStrList = episodes.map(episode =>
    transformTitleToChapterStr(episode.title, episode.bvid)
  );
  return chapterStrList.join('\n');
}

let title = "";
let author = "";
let source = "";
let url = "";
let chapters = "";

if (true) {
  let video_url = await tp.system.prompt("Enter the url of video");
  const video_url_parts = video_url.split('/');
  let bvid = video_url_parts[4];
  bvid = bvid.split('?')[0];

  url = "https://www.bilibili.com/video/" + bvid;
  const api_url = "https://api.bilibili.com/x/web-interface/view?bvid=" + bvid;

  let res;
  try {
    res = await request({ url: api_url, method: "GET" });
  } catch (e) {
    new Notice("API 请求失败，请检查网络或 BV 号");
    throw e;
  }
  const json_res = JSON.parse(res);
  if (!json_res.data) {
    new Notice("API 返回异常，请检查 BV 号");
    throw new Error("No data");
  }

  const data = json_res.data;
  title = data.title;
  author = data.owner.name;

  if ('ugc_season' in data) {
    chapters = get_chapters_from_ugc_season(data);
  } else {
    for (const page of data.pages) {
      chapters += transform_page_to_chapter_str(page, url) + "\n";
    }
  }
  source = "bilibili";
}

const save_folder_path = "/video/";
title = deleteIllegalChars(title);

// ========== 同名文件处理 ==========
// 如果 /video/ 下已存在同名 md，则在文件名后追加当前时间戳
let safeTitle = title;
let counter = 0;
let ext = ".md";
let fullPath;

do {
  fullPath = save_folder_path + safeTitle + ext;
  if (await tp.file.exists(fullPath)) {
    counter++;
    safeTitle = `${title} ${tp.date.now("HHmmss")}`;
  } else {
    break;
  }
} while (counter < 100);

// 移动并改名
await tp.file.move(fullPath);
await tp.file.rename(safeTitle);
// ===================================

-%>
---
title: <% safeTitle %>
author: <% author %>
url: <% url %>
source: <% source %>
score: 
tags: 
type: "video"
date: <% tp.file.creation_date("YYYY-MM-DD") %>
---

## Summary

## Notes
[📺 点击在新窗口播放](<% url %>)

## Review

## Chapters
<% chapters %>

## Reference