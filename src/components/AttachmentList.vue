<script>
import axios from "redaxios";

export default {
  data() {
    return {
      CanAttachmentSend: false,
      CanAttachmentDescription: false,
      CanAttachmentTags: false,
      CanAttachmentDelete: false,
      attachmentList: [],
      tagsList: new Set([]),
      totalResults: 0,
      currentPage: 0,
      totalPages: 0,
      file: null,
      attachment_date_entered: "",
      attachments_tags: "",
      attachments_type: "",
      attachments_name: "",
      isFilterVisible: false,
      isMenuPaginationOpen: false
    };
  },
  setup() {
    function convertDate(date) {
      return new Date(date).toLocaleString("pt-BR").replace(/,/g, " as");
    }

    return {
      convertDate,
    };
  },
  methods: {
    /* getAuth() {
      const url = window.location.origin;
      // Token expirado. Puxa um novo no endpoint e joga passa no header da request atualizado.
      axios
        .get(`${url}/jwt/user-service-token`)
        .then((response) => {
          const USER_TOKEN = response.data;
          const base64Url = USER_TOKEN.split(".")[1];
          const base64 = base64Url.replace(/-/g, "+").replace(/_/g, "/");
          const jsonPayload = decodeURIComponent(
            atob(base64)
              .split("")
              .map(function (c) {
                return "%" + ("00" + c.charCodeAt(0).toString(16)).slice(-2);
              })
              .join("")
          );
          const tokendata = JSON.parse(jsonPayload);
          this.tokenFilter(tokendata);
          axios.defaults.headers.common[
            "Authorization"
          ] = `Bearer ${USER_TOKEN}`;
        })
        .catch((error) => {
          this.error = error;
        });
    }, */
    fetchAttachments() {

      axios
        .get("http://localhost:3000/search")
        .then((response) => {
          this.attachmentList = response.data.attachments;
          this.totalResults = response.data.total_results;
          this.currentPage = response.data.current_page;
          this.totalPages = response.data.total_pages;
          for (let item of this.attachmentList) {
            item.tags.length && item.tags.forEach((tag) => this.tagsList.add(tag));
          }
        })
        .catch((error) => {
          this.error = error;
        });
    },
    filterAttachments() {

      axios
        .get(
          `http://localhost:3000/search?attachment_date_entered=${this.attachment_date_entered}&attachments_name=${this.attachments_name}&attachments_type=${this.attachments_type}&attachments_tags=${this.attachments_tags}`
        )
        .then((response) => {
          this.attachmentList = response.data.attachments;
          this.totalResults = response.data.total_results;
          this.currentPage = response.data.current_page;
          this.totalPages = response.data.total_pages;
        })
        .catch((error) => {
          this.error = error;
        });
    },
    /* loadPage(page) {
      const url = window.location.origin;
      
      axios
      .get(`http://localhost:3000/search`, {
        params: {
          attachment_date_entered: this.attachment_date_entered,
          attachments_name: this.attachments_name,
          attachments_type: this.attachments_type,
          attachments_tags: this.attachments_tags,
          page: page
        }
      })
      .then((response) => {
        this.attachmentList = response.data.attachments;
        this.messagesCount = response.data.total_results;
        this.currentPage = response.data.current_page;
        this.totalPages = response.data.total_pages;
      })
      .catch((error) => {
        this.error = error;
      });
    }, */
    toggleDropdown() {
      this.dropdownOpen = !this.dropdownOpen;
    },
    hideFilters(){
      this.isFilterVisible = false;
    },
    showFilters() {
      this.isFilterVisible = true;
    },
    getUpperMimeType(mimeType) {
      if (mimeType.includes('sticker=true')) {
        return 'Figurinha'.toUpperCase()
      }
      return mimeType.split('/')[1].toUpperCase();
    },
    async getAttachedDownloadLink(attach_id, filename) {
      try {
        const response = await axios.get(`/attachments/attach/download/${attach_id}`, {
          responseType: 'blob' 
        });

        // Cria um link para o arquivo
        const url = window.URL.createObjectURL(new Blob([response.data]));
        const link = document.createElement('a');
        link.href = url;
        link.setAttribute('download', filename);
        link.setAttribute('target', '_blank');
        link.style.display = 'none';
        
        document.body.appendChild(link);
        link.click();
        document.body.removeChild(link);
        window.URL.revokeObjectURL(url);
      } catch (error) {
        console.error('Erro ao obter o link de download:', error);
      }
    },
   
    deleteTag(id, name) {
      for (let item of this.attachmentList) {
        if (item._id.$oid === id) {
          const newTags = item.tags.filter((tag) => tag !== name);
          item.tags = newTags;
          break;
        }
      }
    },
    addTags(id, e) {
      if (!e.target.value) return;

      this.tagsList.add(e.target.value)

      for (let item of this.attachmentList) {
        if (item._id.$oid === id) {
          item.tags.push(e.target.value);
          e.target.value = "";
          break;
        }
      }

    },
    requestTag(id) {
      const formData = new FormData();

      for (let item of this.attachmentList) {
        let arrayTags = [];

        if (item._id.$oid === id) {
          item.tags.forEach((tag) => {
            arrayTags.push(tag);
          });

          formData.append("tags", arrayTags.join(","));
          break;
        }
      }

      const params = new URLSearchParams(formData).toString();
      
      fetch(`${window.location.origin}/attachments/${id}/edit_tags`, {
        method: "POST",
        body: params,
        headers: {
        "Content-Type": "application/x-www-form-urlencoded",
      },
      })
        .catch((error) => {
          console.error("Error:", error);
        });
    },
    toggleMenuPagination(){
      this.isMenuPaginationOpen= !this.isMenuPaginationOpen;
    }
  },
  created() {
    this.fetchAttachments();
  },
};
</script>

<template>
  <div class="card mb-5 shadow">
    <div class="card-header text-primary dash_title header_attachments">
      Seus Arquivos

      <button class="search_attachments_button" @click="showFilters">
        Pesquisar
      </button>
    </div>
    <div class="card-body p-0">
      <form class="attachments_filters_outter_div" v-if="isFilterVisible">
        <div class="attachments_filters">
          <div class="filters">
            <input
              type="date"
              id="attachment_date_entered"
              name="attachment_date_entered"
              v-model="attachment_date_entered"
            />
            <select
              id="attachments_tags"
              name="attachments_tags"
              v-model="attachments_tags"
            >
              <option value="">Selecione uma tag</option>
              <option v-for="(tag, index) in tagsList" :key="index" :value="tag">
                {{ tag }}
              </option>
            </select>
            <select
              id="attachments_type"
              name="attachments_type"
              v-model="attachments_type"
            >
              <option value="">Selecione um tipo</option>
              <option value="attach">Arquivo</option>
              <option value="sticker">Figurinha</option>
            </select>
            <input
              type="text"
              id="attachments_name"
              name="attachments_name"
              v-model="attachments_name"
              placeholder="Nome:"
            />
            <button
              class="filter_button"
              type="submit"
              @click.prevent="filterAttachments"
            >
              Filtrar
            </button>
          </div>

          <button class="button_hide_filters" @click="hideFilters">
            Esconder filtros
            <svg
              width="12"
              height="12"
              viewBox="0 0 12 12"
              fill="none"
              xmlns="http://www.w3.org/2000/svg"
            >
              <path
                d="M3 4.5L6 7.5L9 4.5"
                stroke="#333333"
                stroke-width="0.8"
                stroke-linecap="round"
                stroke-linejoin="round"
              />
            </svg>
          </button>
        </div>
      </form>

      <div class="show_results_attachments">
        <span>
          Exibindo 
          <span v-if="totalPages === 1">{{ totalResults }}</span>
          <span v-else-if="currentPage === totalPages">{{ totalResults - ((totalPages - 1) * 50) }}</span>
          <span v-else>50</span> de {{ totalResults }} resultados.
        </span>
        <div class="pagination_attachments">
          Página

          <div class="box_pagination">
            <button class="button_mais_pags" @click="toggleMenuPagination">
              {{ currentPage }} 
              <svg width="11" height="11" viewBox="0 0 11 11" fill="none" xmlns="http://www.w3.org/2000/svg">
              <path d="M3.23242 4.36719L5.49909 6.63385L7.76576 4.36719" stroke="#333333" stroke-opacity="0.5" stroke-linecap="round" stroke-linejoin="round"/>
              </svg>
            </button>
            <ul class="menu_pagination" v-show="isMenuPaginationOpen">
              <template v-for="page in totalPages" :key="page">
                <li :class="{'active': currentPage === page}" v-if="(page >= currentPage && page <= currentPage + pageSpacer) || (page <= currentPage && page >= currentPage - pageSpacer)">
                <a class="page-link" @click="loadPage(page)">{{ page }}</a>
                </li>
                <li v-else-if="page === 1 || page === totalPages" class="page-item">
                  <a class="page-link" @click="loadPage(page)">
                    {{ page }}
                  </a>
                </li>
                <li v-else>
                  <span class="page-link" href="#">...</span>
                </li>
              </template>
            </ul>
          </div>
          <button class="botao-voltar" @click="loadPage(currentPage - 1)" :disabled="currentPage === 1">
            <i class="uil uil-arrow-left"></i>
          </button>

          <button class="botao-avancar" @click="loadPage(currentPage + 1)" :disabled="currentPage === totalPages">
            <i class="uil uil-arrow-right"></i>
          </button>
          
        </div>
      </div>
      

      <table class="table">
        <thead>
          <tr class="header_attachments_results">
            <th></th>
            <th>
              <a href="/attachments/1/name">NOME DO ARQUIVO</a>
            </th>
            <th>
              <a href="/attachments/1/tags">TAGS</a>
            </th>
            <th>
              <a href="/attachments/1/created">CADASTRO</a>
            </th>
            <th></th>
          </tr>
        </thead>
        <tbody class="content_attachments">
          <tr v-for="(file, index) in attachmentList" :key="index">
            <td class="align-middle text-center">
              <a
                :href="file.path_relative"
                target="_blank"
                class="lightbox-modal"
              >
                <img
                  :src="file.thumbnail"
                  alt=""
                  height="60"
                  width="60"
                  class="rounded"
                />
              </a>
            </td>
            <td class="text-nowrap align-middle">
              <a style="cursor: pointer">
                {{ file.original_name }}
              </a>
              <span class="badge badge-info">{{ getUpperMimeType(file.mime) }}</span>
              <textarea
                v-if="CanAttachmentDescription"
                :data-attachment-id="file._id.$oid"
                placeholder="Legenda:"
                v-model="file.description"
                rows="1"
                name=""
                class="attachment_edit form-control-sm form-control"
              ></textarea>
              <textarea
                v-else
                class="attachment_edit form-control-sm form-control"
                readonly
              ></textarea>
            </td>
            <td class="align-middle">
              <div class="tag__wrapper">
                <span
                  class="tag__tags"
                  v-for="(tag, index) in file.tags"
                  :key="index"
                >
                  {{ tag }}
                  <span
                    class="tag__close"
                    @click="
                      deleteTag(file._id.$oid, tag), requestTag(file._id.$oid)
                    "
                    >X</span
                  >
                </span>
                <input
                  @keyup.enter="
                    addTags(file._id.$oid, $event), requestTag(file._id.$oid)
                  "
                  type="text"
                  class="tag__input"
                  data-role="tagsinput"
                  placeholder="Tags:"
                  :data-attach-id="file.id"
                  value=""
                />
              </div>
            </td>
            <td class="text-nowrap align-middle">
              {{ convertDate(file.created.$date) }}
            </td>
            <td v-if="CanAttachmentDelete" class="align-middle">
              <button
                class="attachment_delete"
                :data-attachment-id="file._id.$oid"
              >
                <svg
                  width="16"
                  height="16"
                  viewBox="0 0 16 16"
                  fill="none"
                  xmlns="http://www.w3.org/2000/svg"
                >
                  <g clip-path="url(#clip0_3246_96)">
                    <path
                      d="M2.05664 3.11475H3.42826H14.4012"
                      stroke="white"
                      stroke-width="1.37162"
                      stroke-linecap="round"
                      stroke-linejoin="round"
                    />
                    <path
                      d="M5.48712 3.11531V1.74369C5.48712 1.37992 5.63163 1.03104 5.88886 0.773809C6.14609 0.51658 6.49497 0.37207 6.85874 0.37207H9.60199C9.96576 0.37207 10.3146 0.51658 10.5719 0.773809C10.8291 1.03104 10.9736 1.37992 10.9736 1.74369V3.11531M13.031 3.11531V12.7167C13.031 13.0804 12.8865 13.4293 12.6293 13.6865C12.3721 13.9438 12.0232 14.0883 11.6594 14.0883H4.80131C4.43753 14.0883 4.08866 13.9438 3.83143 13.6865C3.5742 13.4293 3.42969 13.0804 3.42969 12.7167V3.11531H13.031Z"
                      stroke="white"
                      stroke-width="1.37162"
                      stroke-linecap="round"
                      stroke-linejoin="round"
                    />
                    <path
                      d="M6.85742 6.54443V10.6593"
                      stroke="white"
                      stroke-width="1.37162"
                      stroke-linecap="round"
                      stroke-linejoin="round"
                    />
                    <path
                      d="M9.60156 6.54443V10.6593"
                      stroke="white"
                      stroke-width="1.37162"
                      stroke-linecap="round"
                      stroke-linejoin="round"
                    />
                  </g>
                  <defs>
                    <clipPath id="clip0_3246_96">
                      <rect width="16" height="16" fill="white" />
                    </clipPath>
                  </defs>
                </svg>
              </button>
            </td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>

<style lang="scss">
.tag {
  &__wrapper {
    border: 1px solid #ccc;
    box-shadow: inset 0 1px 1px rgba(0, 0, 0, 0.075);
    display: flex;
    flex-direction: row;
    gap: 5px;
    padding: 5px;
    border-radius: 3px;
    max-width: 275px;
    width: 100%;
    font-size: 12px;
    flex-wrap: wrap;
    overflow: auto;
    height: 36px;
  }

  &__tags {
    border: 1px solid #01649e;
    padding-inline: 5px;
    background-color: #36b9cc;
    border-radius: 3px;
    font-size: 75%;
    color: #fff;
    display: flex;
    gap: 10px;
    align-items: center;
    justify-content: center;
  }

  &__close {
    width: 18px;
    height: 18px;
    background-color: darken(#36b9cc, 10%);
    display: grid;
    place-items: center;
    border-radius: 100%;
    font-size: 10px;
    cursor: pointer;

    &:hover {
      background-color: darken(#36b9cc, 30%);
    }
  }

  &__input {
    width: min-content;
    border: none;
    background-color: transparent;
    flex: 1;
    padding-left: 10px;

    &::placeholder {
      font-size: 15px;
    }
    &:focus-visible{
      outline: none;
    }
  }
}
</style>
