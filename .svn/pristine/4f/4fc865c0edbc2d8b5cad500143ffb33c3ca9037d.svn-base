#!/usr/bin/env python
# -*- coding: utf-8 -*-
"""
WRE 운영 반영 스크립트
로컬 WebContent → SFTP(192.168.100.216) → Docker 반영

사용법:
  python deploy_sftp.py                    # 전체 전송
  python deploy_sftp.py guide              # guide 폴더만 전송
  python deploy_sftp.py _wpack_            # 빌드된 JS 전체 전송
  python deploy_sftp.py _wpack_/guide      # guide의 빌드 JS만 전송
  python deploy_sftp.py --js               # _wpack_ JS만 전송 (--js 단축키)
  python deploy_sftp.py --js guide         # guide의 빌드 JS만 전송
  python deploy_sftp.py --force guide      # guide 폴더 강제 전송 (크기 비교 무시)
  python deploy_sftp.py --file page/P00441.xml  # 단일 파일 전송
"""

import os
import sys
import paramiko
from datetime import datetime

# === 설정 ===
SFTP_HOST = "192.168.100.216"
SFTP_PORT = 22
SFTP_USER = "wtech_admin"
SFTP_PASS = "WTECHadmin5!"
REMOTE_ROOT = "/Users/wtech_admin/webapp/ROOT"
LOCAL_WEBCONTENT = os.path.join(os.path.dirname(os.path.abspath(__file__)), "WebContent")
DOCKER_CONTAINER = "wtech_admin_container"
DOCKER_WEBAPPS = "/usr/local/tomcat/webapps/ROOT"
DOCKER_BIN = "/Applications/Docker.app/Contents/Resources/bin/docker"


def log(msg):
    timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    print(f"[{timestamp}] {msg}")


def sftp_mkdir_p(sftp, remote_dir):
    """원격 디렉토리를 재귀적으로 생성"""
    dirs_to_create = []
    current = remote_dir
    while True:
        try:
            sftp.stat(current)
            break
        except FileNotFoundError:
            dirs_to_create.append(current)
            current = os.path.dirname(current)
            if current == "/" or current == "":
                break
    for d in reversed(dirs_to_create):
        try:
            sftp.mkdir(d)
        except Exception:
            pass


def upload_directory(sftp, local_dir, remote_dir, force=False, uploaded_count=None):
    """로컬 디렉토리를 SFTP로 업로드"""
    if uploaded_count is None:
        uploaded_count = [0]

    sftp_mkdir_p(sftp, remote_dir)

    for item in os.listdir(local_dir):
        local_path = os.path.join(local_dir, item)
        remote_path = remote_dir + "/" + item

        if item.startswith(".git"):
            continue

        if os.path.isdir(local_path):
            upload_directory(sftp, local_path, remote_path, force, uploaded_count)
        else:
            try:
                if not force:
                    local_size = os.path.getsize(local_path)
                    try:
                        remote_stat = sftp.stat(remote_path)
                        if local_size == remote_stat.st_size:
                            continue
                    except FileNotFoundError:
                        pass

                sftp.put(local_path, remote_path)
                sftp.chmod(remote_path, 0o755)
                uploaded_count[0] += 1
                log(f"  전송: {remote_path}")
            except Exception as e:
                log(f"  오류: {remote_path} - {e}")

    return uploaded_count[0]


def upload_single_file(sftp, local_file, remote_file):
    """단일 파일 SFTP 업로드"""
    remote_dir = os.path.dirname(remote_file)
    sftp_mkdir_p(sftp, remote_dir)
    sftp.put(local_file, remote_file)
    sftp.chmod(remote_file, 0o755)
    log(f"  전송: {remote_file}")


def docker_cp(ssh, remote_path, docker_path):
    """운영 서버에서 Docker 컨테이너로 복사"""
    docker_cmd = f"{DOCKER_BIN} cp {remote_path} {DOCKER_CONTAINER}:{docker_path}"
    stdin, stdout, stderr = ssh.exec_command(docker_cmd)
    exit_code = stdout.channel.recv_exit_status()
    err = stderr.read().decode().strip()
    if exit_code == 0:
        log("[3/3] Docker 반영 완료!")
    else:
        log(f"[3/3] Docker 반영 오류: {err}")
    return exit_code == 0


def main():
    args = sys.argv[1:]
    force = False
    js_mode = False
    file_mode = False
    sub_path = None

    # 옵션 파싱
    while args and args[0].startswith("--"):
        opt = args.pop(0)
        if opt == "--force":
            force = True
        elif opt == "--js":
            js_mode = True
        elif opt == "--file":
            file_mode = True

    if args:
        sub_path = args[0].replace("\\", "/")

    # --js 모드: _wpack_ 경로 자동 설정
    if js_mode:
        if sub_path:
            sub_path = f"_wpack_/{sub_path}"
        else:
            sub_path = "_wpack_"

    # --file 모드: 단일 파일 전송
    if file_mode:
        if not sub_path:
            log("오류: --file 옵션에는 파일 경로가 필요합니다")
            log("예시: python deploy_sftp.py --file page/P00441.xml")
            sys.exit(1)
        local_file = os.path.join(LOCAL_WEBCONTENT, sub_path)
        remote_file = REMOTE_ROOT + "/" + sub_path
        if not os.path.isfile(local_file):
            log(f"오류: 파일이 없습니다 - {local_file}")
            sys.exit(1)

        log(f"단일 파일 전송: {sub_path}")
        ssh = paramiko.SSHClient()
        ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
        ssh.connect(SFTP_HOST, port=SFTP_PORT, username=SFTP_USER, password=SFTP_PASS,
                    timeout=30, allow_agent=False, look_for_keys=False)
        sftp = ssh.open_sftp()
        upload_single_file(sftp, local_file, remote_file)
        sftp.close()

        # Docker 반영
        log("[3/3] Docker 반영 중...")
        docker_remote = REMOTE_ROOT + "/" + sub_path
        docker_dest = DOCKER_WEBAPPS + "/" + sub_path
        docker_cp(ssh, docker_remote, docker_dest)
        ssh.close()
        log("=== 배포 완료! ===")
        return

    # 디렉토리 전송 모드
    if sub_path:
        local_dir = os.path.join(LOCAL_WEBCONTENT, sub_path)
        remote_dir = REMOTE_ROOT + "/" + sub_path
        if not os.path.exists(local_dir):
            log(f"오류: 경로가 없습니다 - {local_dir}")
            sys.exit(1)
        mode_name = "JS 전송" if js_mode else "부분 전송"
        log(f"{mode_name} 모드: {sub_path}" + (" (강제)" if force else ""))
    else:
        local_dir = LOCAL_WEBCONTENT
        remote_dir = REMOTE_ROOT
        log("전체 전송 모드" + (" (강제)" if force else ""))

    # SSH/SFTP 연결
    log(f"[1/3] SFTP 접속 중... {SFTP_HOST}")
    ssh = paramiko.SSHClient()
    ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    ssh.connect(SFTP_HOST, port=SFTP_PORT, username=SFTP_USER, password=SFTP_PASS,
                timeout=30, allow_agent=False, look_for_keys=False)
    sftp = ssh.open_sftp()
    log("[1/3] SFTP 접속 성공")

    # 파일 전송
    log("[2/3] 파일 전송 시작...")
    count = upload_directory(sftp, local_dir, remote_dir, force)
    log(f"[2/3] 파일 전송 완료 (변경된 파일 {count}개)")
    sftp.close()

    # Docker 반영
    log("[3/3] Docker 반영 중...")
    if sub_path:
        docker_cp(ssh, f"{REMOTE_ROOT}/{sub_path}/.", f"{DOCKER_WEBAPPS}/{sub_path}/")
    else:
        docker_cp(ssh, f"{REMOTE_ROOT}/.", f"{DOCKER_WEBAPPS}/")

    ssh.close()
    log("=== 배포 완료! ===")


if __name__ == "__main__":
    main()
