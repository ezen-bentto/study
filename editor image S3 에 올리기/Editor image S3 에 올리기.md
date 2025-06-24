# Editor image S3 에 올리기

## 개요

1. 필요한 이유

2. 서비스 진행

3. 코드

## 1. 필요한 이유

quill, toastUI 등의 Editor 로 image 를 업로드 할 경우
2가지 방법이 있다.

1. base64 형식으로 저장

2. S3 에 저장하고 해당 url 을 받아서 저장

각 방식의 장단점이 있는데

### Base64 형식으로 저장

- 장점

  - 별도의 서버나 스토리지 설정 없이 이미지 데이터를 바로 에디터에 삽입 가능

  - 전체 콘텐츠가 하나의 HTML 혹은 JSON에 포함되므로 전송이 간편함

- 단점

  - 이미지가 텍스트 데이터보다 용량이 크므로, 전체 콘텐츠 크기가 커짐 → DB 및 전송 속도에 부하 발생 -> base64 인코딩 시 원본 이미지보다 약 33% 용량 증가함

  - 반복 조회 시 성능 저하 우려 (특히 목록 페이지에서 미리보기 등을 로딩할 경우)

  - 캐싱 불가능 → 매번 전체 데이터를 다시 불러옴

  - 콘텐츠 내 여러 이미지가 있을 경우, 렌더링이 느려질 수 있음

### S3에 저장하고 URL로 참조

- 장점

  이미지와 콘텐츠를 분리해 관리할 수 있음 → 콘텐츠 저장 및 전송 용량 최소화

  이미지 파일은 AWS S3에 저장되므로 클라우드 스토리지를 활용한 효율적인 관리 가능

  URL 기반 접근으로 브라우저 캐싱 적용 가능 → 페이지 로딩 속도 개선

  이미지 재사용 및 관리(예: 삭제, 갱신) 용이

- 단점

  업로드 API 개발 필요 (백엔드 연동)

  AWS S3 권한 및 보안 설정 고려해야 함

  업로드 중 에러 처리

> 우리 프로젝트 경우 이미 S3 가 연결 되어 있고, 에디터에 이미지 업로드 하면 그걸 따로 저장 해놨다가 카드 썸네일, 상세페이지 이미지 등 별도로 사용할 생각으로 S3 를 채택함

> 그리고 지피티한태 물어보니까 S3 추천함 ㅎㅎ

## 2. 서비스 진행

1. 사용자가 글 작성 중 이미지 첨부

   Quill 또는 Toast UI Editor 내에서 이미지 삽입 버튼 클릭

2. 프론트엔드가 이미지 선업로드 요청

   이미지 선택 시, 게시글 저장 전에 우선 이미지만 서버에 전송

3. 프론트엔드에서 이미지 파일을 FormData 형태로 /upload API에 전송

   FormData.append("file", file) 형식

4. Express 서버는 S3에 이미지를 업로드하고, 업로드된 이미지의 S3 URL을 응답

   예: https://your-bucket.s3.amazonaws.com/editor/image123.png

5. 프론트는 응답받은 S3 이미지 URL을 에디터에 삽입

   이미지가 `<img src="S3 URL">` 형식으로 에디터 본문에 포함됨

6. 사용자가 글 작성 완료 후, 제목/내용과 함께 /contest 등의 API에 POST 요청

7. 백엔드는 해당 데이터를 DB에 저장

## 3. 코드

- 프론트 form-data

  ```ts
  // components/ReactQuillEditor.tsx
  import ReactQuill from 'react-quill-new';
  import Quill from 'quill';
  import { fetchUpload } from '@/api/common/upload';

  // 서버에 이미지 업로드 요청하는 함수
  const uploadImage = async (file: File): Promise<string> => {
    const formData = new FormData();
    formData.append('image', file);

    const res = await fetchUpload(formData);

    return res.data.url; // S3 이미지 URL
  };

  // Quill 툴바 설정 + 이미지 핸들러 포함
  const createEditorModules = () => ({
    toolbar: {
      container: [
        [{ header: [1, 2, false] }],
        ['bold', 'italic', 'underline', 'strike', 'blockquote'],
        [{ list: 'ordered' }, { list: 'bullet' }, { indent: '-1' }, { indent: '+1' }],
        ['link', 'image'],
        ['clean'],
      ],
      handlers: {
        // eslint-disable-next-line no-unused-vars
        image: function (this: Quill) {
          const input = document.createElement('input');
          input.setAttribute('type', 'file');
          input.setAttribute('accept', 'image/*');
          input.click();

          input.onchange = async () => {
            const file = input.files?.[0];
            if (!file) return;
            console.info(file);

            try {
              const url = await uploadImage(file);
              const range = this.getSelection(true);
              this.insertEmbed(range.index, 'image', url);
              this.setSelection(range.index + 1);
            } catch (err) {
              console.error('이미지 업로드 실패:', err);
            }
          };
        },
      },
    },
  });

  type Props = {
    value: string;
    // eslint-disable-next-line no-unused-vars
    onChange: (value: string) => void;
    className?: string;
  };

  const ReactQuillEditor = ({ value, onChange, className }: Props) => {
    return (
      <ReactQuill
        value={value}
        onChange={onChange}
        theme="snow"
        placeholder="내용을 입력하세요..."
        modules={createEditorModules()}
        className={className}
      />
    );
  };

  export default ReactQuillEditor;
  ```

- 백 service

  ```ts
  // src/service/upload.service.ts
  import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
  import { v4 as uuidv4 } from 'uuid';
  import path from 'path';
  import { ENV } from '@/config/env.config';
  import sharp from 'sharp';

  const s3 = new S3Client({
    region: ENV.aws.region!,
    credentials: {
      accessKeyId: ENV.aws.accessKeyId!,
      secretAccessKey: ENV.aws.secretAccessKey!,
    },
  });

  export const UploadService = {
    uploadImageToS3: async (file: Express.Multer.File): Promise<string> => {
      const ext = path.extname(file.originalname).toLowerCase();
      const isImage = ['.jpg', '.jpeg', '.png', '.webp'].includes(ext);

      if (!isImage) {
        throw new Error('지원하지 않는 이미지 형식입니다.');
      }

      // ✅ sharp 적용: 800px 리사이즈, WebP 변환, 압축
      const optimizedBuffer = await sharp(file.buffer)
        .resize({ width: 800, withoutEnlargement: true }) // 원본보다 크면 그대로
        .webp({ quality: 75 }) // WebP 압축 품질 설정
        .toBuffer();

      const key = `uploads/${uuidv4()}.webp`;

      const command = new PutObjectCommand({
        Bucket: ENV.aws.bucket!,
        Key: key,
        Body: optimizedBuffer,
        ContentType: 'image/webp',
      });

      await s3.send(command);

      return `https://${ENV.aws.bucket}.s3.${ENV.aws.region}.amazonaws.com/${key}`;
    },
  };
  ```

이렇게 각각 받아지는걸 확인하고 실제 s3 에 저장되는 것을 확인해야하나

여기까지~ 나머지는 알잘딱 하도록~
